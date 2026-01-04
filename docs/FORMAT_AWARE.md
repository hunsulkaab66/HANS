# Format-Aware Optimization

This document describes how HANS — Hardware-Aware Neural Storage — exploits
knowledge of AI data formats to optimize caching, chunking, and prefetching.

Unlike general-purpose filesystems, HANS understands the internal structure of
common AI data formats and uses this knowledge to make smarter decisions.

---

## Why Format Awareness Matters

AI datasets are not generic blobs. They have structure:

- **Columnar formats** (Parquet, Arrow): Allow skipping irrelevant columns
- **Sequential formats** (TFRecord, RecordIO): Have predictable access patterns
- **Tensor formats** (Safetensors, PyTorch .pt): Contain embedded metadata
- **Image formats** (JPEG, PNG): Can be decoded directly to GPU

By understanding these formats, HANS can:
- Prefetch only the data that will actually be accessed
- Chunk data at natural boundaries (records, not arbitrary offsets)
- Decode or decompress data in-flight
- Skip unnecessary I/O entirely

---

## Supported Formats

HANS provides format-aware optimizations for:

### Tier 1 (Fully Supported)
- **Parquet**: Columnar data format
- **Arrow IPC**: In-memory columnar format
- **TFRecord**: TensorFlow's sequential record format
- **Safetensors**: Emerging PyTorch tensor format
- **HDF5**: Hierarchical tensor storage

### Tier 2 (Partial Support)
- **PyTorch .pt / .pth**: Pickle-based tensor serialization
- **NumPy .npy / .npz**: Binary array format
- **CSV / TSV**: Text-based tabular data
- **JPEG / PNG**: Compressed images (detection only)

### Tier 3 (Detection Only)
- **AVRO**: Data serialization format
- **ORC**: Optimized row columnar format
- **Feather**: Arrow-based file format

Formats not listed are treated as opaque binary blobs.

---

## Format Detection

HANS detects formats via:

1. **File extension** (fast, first pass)
2. **Magic bytes** (reliable, second pass)
3. **Metadata hints** (from PyTorch DataLoader, etc.)

### Detection Pipeline

```cpp
FileFormat detectFormat(const std::string& path) {
    // Fast path: check extension
    FileFormat fmt = detectByExtension(path);
    if (fmt != FileFormat::UNKNOWN) {
        return fmt;
    }
    
    // Slow path: read magic bytes
    std::vector<uint8_t> header = readHeader(path, 64);
    return detectByMagic(header);
}
```

### Magic Byte Signatures

| Format | Magic Bytes (hex) | Offset |
|--------|-------------------|--------|
| Parquet | `50 41 52 31` ("PAR1") | 0 |
| Arrow IPC | `FF FF FF FF` (continuation) | 0 |
| HDF5 | `89 48 44 46 0D 0A 1A 0A` | 0 |
| TFRecord | No fixed magic (CRC-based) | - |
| Safetensors | JSON header length | 0-8 |
| PNG | `89 50 4E 47` | 0 |
| JPEG | `FF D8 FF` | 0 |

---

## Format-Specific Optimizations

### Parquet

#### Structure

Parquet files consist of:
- **Footer**: Column metadata, row group info
- **Row Groups**: Horizontal data partitions
- **Column Chunks**: Vertical data within row groups
- **Pages**: Smallest unit of I/O

#### HANS Optimizations

**1. Footer-First Loading**

Parquet footers are at the end of the file. HANS prefetches the footer first:

```cpp
void prefetchParquet(const std::string& path) {
    // Read last 8 bytes to get footer size
    uint32_t footerSize = readFooterSize(path);
    
    // Prefetch footer into cache
    size_t footerOffset = fileSize - footerSize - 8;
    prefetchRange(path, footerOffset, footerSize);
}
```

**2. Column Pruning**

If PyTorch DataLoader requests specific columns, HANS skips unrequested
column chunks entirely:

```cpp
// User requests only columns: ["image", "label"]
ParquetMetadata meta = parseFooter(path);

for (ColumnChunk& chunk : meta.rowGroups[0].columns) {
    if (chunk.name == "image" || chunk.name == "label") {
        prefetchRange(path, chunk.offset, chunk.size);
    }
    // Skip other columns
}
```

**3. Row Group Alignment**

HANS aligns cache chunks with Parquet row group boundaries to avoid
splitting logical records across chunks.

---

### Arrow IPC (Feather)

#### Structure

Arrow IPC files are:
- Self-describing (schema embedded)
- Memory-mappable
- Zero-copy when possible

#### HANS Optimizations

**1. Zero-Copy Access**

For Arrow IPC files in RAM tier, HANS exposes them via mmap-like interface:

```cpp
ArrowBuffer mapArrowFile(const std::string& path) {
    // File already cached in RAM
    void* cached = getCachedBuffer(path);
    
    // Return zero-copy Arrow buffer
    return ArrowBuffer::wrap(cached, fileSize);
}
```

**2. Schema-Aware Prefetch**

Arrow schemas describe column types and sizes. HANS uses this to predict
access patterns:

```cpp
void prefetchArrowColumns(const Schema& schema, 
                          const std::vector<std::string>& columns) {
    for (const std::string& col : columns) {
        size_t colSize = schema.getColumnSize(col);
        // Prefetch with appropriate priority
    }
}
```

---

### TFRecord

#### Structure

TFRecord is a sequential format:
```
[length][length_crc][data][data_crc]
[length][length_crc][data][data_crc]
...
```

#### HANS Optimizations

**1. Sequential Prefetch**

TFRecord access is highly sequential. HANS uses aggressive sequential
prefetch:

```cpp
void prefetchTFRecord(const std::string& path, size_t offset) {
    // Read length of next record
    uint64_t length = readRecordLength(path, offset);
    
    // Prefetch entire record + next N records
    size_t prefetchSize = length + LOOKAHEAD_RECORDS * AVG_RECORD_SIZE;
    prefetchRange(path, offset, prefetchSize);
}
```

**2. Record Boundary Awareness**

HANS never splits TFRecord records across cache chunks. Chunks always end
at record boundaries:

```cpp
size_t alignToRecordBoundary(const std::string& path, size_t offset) {
    // Scan forward to find next record start
    while (!isRecordStart(path, offset)) {
        offset++;
    }
    return offset;
}
```

---

### Safetensors

#### Structure

Safetensors files contain:
- 8-byte header: JSON metadata length
- JSON metadata: Tensor names, shapes, dtypes, offsets
- Raw tensor data

#### HANS Optimizations

**1. Metadata-First Loading**

Safetensors metadata is at the beginning. HANS loads it first:

```cpp
void prefetchSafetensors(const std::string& path) {
    // Read JSON length
    uint64_t jsonLen = readUint64(path, 0);
    
    // Prefetch and parse metadata
    std::string jsonMeta = readRange(path, 8, jsonLen);
    SafetensorsMetadata meta = parseMetadata(jsonMeta);
    
    // Now selectively prefetch tensors
}
```

**2. Selective Tensor Loading**

If the model only needs certain tensors (e.g., "layer1.weight"), HANS
prefetches only those:

```cpp
void prefetchTensor(const SafetensorsMetadata& meta,
                    const std::string& tensorName) {
    TensorInfo info = meta.getTensor(tensorName);
    
    // Prefetch only this tensor's data
    size_t offset = 8 + meta.headerSize + info.offset;
    prefetchRange(path, offset, info.size);
}
```

**3. Zero-Copy Tensor Views**

When Safetensors files are cached in pinned memory, HANS can return zero-copy
PyTorch tensors:

```cpp
torch::Tensor loadTensorZeroCopy(const std::string& path,
                                 const std::string& tensorName) {
    void* cached = getCachedBuffer(path);
    TensorInfo info = meta.getTensor(tensorName);
    
    // Return PyTorch tensor wrapping cached memory
    return torch::from_blob(
        cached + info.offset,
        info.shape,
        info.dtype
    );
}
```

---

### HDF5

#### Structure

HDF5 is a hierarchical format:
- Groups (like directories)
- Datasets (multi-dimensional arrays)
- Attributes (metadata)

#### HANS Optimizations

**1. Chunked Dataset Awareness**

HDF5 datasets are often stored in chunks. HANS aligns cache chunks with
HDF5 internal chunks:

```cpp
void prefetchHDF5Dataset(const std::string& path,
                         const std::string& dataset) {
    HDF5Metadata meta = parseHDF5(path);
    DatasetInfo info = meta.getDataset(dataset);
    
    // Prefetch only accessed HDF5 chunks
    for (ChunkCoord coord : info.accessedChunks) {
        size_t offset = info.getChunkOffset(coord);
        prefetchRange(path, offset, info.chunkSize);
    }
}
```

**2. Metadata Caching**

HDF5 metadata access can be slow. HANS caches parsed metadata separately:

```cpp
struct HDF5MetadataCache {
    std::unordered_map<std::string, HDF5Metadata> cache;
    
    HDF5Metadata get(const std::string& path) {
        if (!cache.contains(path)) {
            cache[path] = parseHDF5(path);
        }
        return cache[path];
    }
};
```

---

### PyTorch .pt / .pth

#### Structure

PyTorch files use Python's pickle format:
- Unpredictable internal structure
- Opaque to non-Python parsers

#### HANS Optimizations

**1. Whole-File Caching**

Since .pt files are opaque, HANS treats them as single units:

```cpp
void prefetchPyTorchCheckpoint(const std::string& path) {
    // Prefetch entire file
    prefetchRange(path, 0, fileSize(path));
}
```

**2. Checkpoint Pinning**

PyTorch checkpoints are write-heavy during training. HANS pins them in cache:

```cpp
void pinCheckpoint(const std::string& path) {
    CacheEntry* entry = getEntry(path);
    entry->pinned = true;
    entry->evictionPriority = NEVER_EVICT;
}
```

---

### Image Formats (JPEG, PNG)

#### Structure

Compressed raster images.

#### HANS Optimizations

**1. Batch Prefetch**

Image datasets (ImageNet, etc.) are accessed in batches. HANS prefetches
entire batches:

```cpp
void prefetchImageBatch(const std::vector<std::string>& paths) {
    for (const std::string& path : paths) {
        // Small files, prefetch aggressively
        prefetchRange(path, 0, fileSize(path));
    }
}
```

**2. GPU Decode Offload (Future)**

HANS could integrate with nvJPEG to decode images directly on GPU:

```cpp
// Future work
void decodeToGPU(const std::string& path, void* gpuBuffer) {
    nvjpegImage_t output;
    nvjpegDecode(handle, decoder, cached, fileSize, 
                 NVJPEG_OUTPUT_RGB, &output, stream);
}
```

---

## Format-Aware Chunking

HANS adjusts chunk sizes based on format:

| Format | Default Chunk Size | Reason |
|--------|-------------------|--------|
| Parquet | Row group size | Natural boundary |
| Arrow IPC | Record batch size | Memory-aligned |
| TFRecord | 10-100 records | Sequential access |
| Safetensors | Per-tensor | Independent tensors |
| HDF5 | Dataset chunk size | Internal chunking |
| .pt | Whole file | Opaque format |
| JPEG | 1-10 images | Small files |

Chunking at natural boundaries:
- Reduces wasted I/O
- Enables format-specific optimizations
- Improves cache hit rates

---

## Format-Aware Prefetching

Different formats have different access patterns:

### Sequential Formats (TFRecord, CSV)
- **Pattern**: Linear scan
- **Prefetch**: Large, aggressive lookahead

### Columnar Formats (Parquet, Arrow)
- **Pattern**: Vertical slicing (columns)
- **Prefetch**: Column-aware, metadata-driven

### Hierarchical Formats (HDF5, Zarr)
- **Pattern**: Sparse, dataset-specific
- **Prefetch**: Dataset-level, user-hinted

### Random Access (Safetensors, .pt)
- **Pattern**: Unpredictable
- **Prefetch**: Conservative, whole-file

---

## Format Detection API

HANS exposes format hints via client API:

```python
# Python API
import hans

# Explicit format hint
loader = hans.DataLoader(
    path="dataset.parquet",
    format="parquet",
    columns=["image", "label"]  # Only prefetch these columns
)

# Automatic detection
loader = hans.DataLoader(
    path="dataset.tfrecord",
    format="auto"  # HANS detects TFRecord
)
```

---

## Limitations

Format awareness is best-effort:

- HANS cannot parse every variant of every format
- Corrupted files fall back to opaque handling
- Custom or proprietary formats are unsupported
- Format parsing adds CPU overhead (minimized via caching)

When in doubt, HANS falls back to treating files as opaque binary data.

---

## Future Format Support

Planned additions:

- **Zarr**: Chunked N-dimensional arrays
- **Lance**: Columnar format for ML
- **MessagePack**: Binary serialization
- **Protocol Buffers**: Structured data
- **AVRO**: Full support (currently detection-only)

Format support will be added based on real-world usage patterns.

---

## Performance Impact

Format awareness provides measurable benefits:

### Parquet Column Pruning
- Baseline (ext4): Read entire file
- HANS: Read only requested columns
- Speedup: 2-5x (depending on schema)

### TFRecord Sequential Prefetch
- Baseline: Reactive, per-record I/O
- HANS: Aggressive sequential prefetch
- Speedup: 1.5-3x (I/O bound workloads)

### Safetensors Selective Loading
- Baseline: Load entire file
- HANS: Load only required tensors
- Speedup: 1.2-10x (depends on model size vs. needed tensors)

Benchmarks are provided in `docs/BENCHMARKS.md`.

---

## Summary

Format awareness transforms HANS from a generic cache into an intelligent,
AI-native storage layer.

By understanding the internal structure of AI data formats, HANS:
- Reduces unnecessary I/O
- Improves prefetch accuracy
- Enables zero-copy optimizations
- Aligns caching with logical data boundaries

Format awareness is a core differentiator and will continue to evolve as new
AI data formats emerge.
