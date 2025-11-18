# Phân Tích Mã Nguồn FFMediaElement
## Cơ Chế Giải Mã Video và Phát Trên GPU

**Dự án:** ffmediaelement (https://github.com/unosquare/ffmediaelement)
**Phiên bản FFmpeg:** 7.0
**Ngôn ngữ:** C# (.NET 8.0 / .NET Framework 4.8)
**Tác giả phân tích:** AI Assistant
**Ngày:** 2025-11-18

---

## 1. TỔNG QUAN DỰ ÁN

### 1.1. Giới thiệu
FFMediaElement (FFME) là một control WPF Media Player tiên tiến sử dụng FFmpeg để giải mã và phát media, thay thế cho MediaElement chuẩn của WPF (sử dụng DirectShow). Dự án được viết bằng C# và sử dụng P/Invoke để tích hợp với thư viện FFmpeg native thông qua FFmpeg.AutoGen.

### 1.2. Kiến trúc 3 lớp

```
┌─────────────────────────────────────────────────────────────────┐
│                        MediaElement (WPF)                        │
│          - Platform-specific implementation                      │
│          - Property synchronization                              │
│          - Event handling                                        │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                         MediaEngine                              │
│          - Command management (Play/Pause/Seek/Stop)             │
│          - 3 background workers:                                 │
│            1. PacketReadingWorker  (Đọc packets)                 │
│            2. FrameDecodingWorker  (Decode frames)               │
│            3. BlockRenderingWorker (Render blocks)               │
└────────────────────────┬────────────────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────────────────┐
│                      MediaContainer                              │
│          - Input stream management                               │
│          - Packet caching                                        │
│          - Frame decoding                                        │
│          - Block conversion                                      │
│          - Component set (Audio/Video/Subtitle)                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3. Pipeline xử lý dữ liệu

```
Input Stream → Packets → Frames → Blocks → Renderer → Screen/Speakers
     ↓            ↓         ↓        ↓         ↓
  Container    Cache   Decoder  Converter  Platform
                                           Specific
```

**Giải thích:**
- **Packets**: Dữ liệu nén đọc từ input stream
- **Frames**: Dữ liệu thô sau khi decode (AVFrame của FFmpeg)
- **Blocks**: Dữ liệu đã được convert sang format chuẩn (BGRA cho video)

---

## 2. CƠ CHẾ GIẢI MÃ VIDEO

### 2.1. VideoComponent - Thành phần chính

**File:** `Unosquare.FFME/Container/VideoComponent.cs`

VideoComponent là lớp chịu trách nhiệm chính trong việc giải mã video. Nó kế thừa từ `MediaComponent` và cung cấp các chức năng:

#### Thuộc tính quan trọng:
```csharp
public int FrameWidth { get; }           // Chiều rộng frame
public int FrameHeight { get; }          // Chiều cao frame
public double BaseFrameRate { get; }     // Frame rate gốc
public double AverageFrameRate { get; }  // Frame rate trung bình
public HardwareAccelerator HardwareAccelerator { get; } // Hardware decoder
public bool IsUsingHardwareDecoding { get; } // Trạng thái HW decode
```

#### Các biến quan trọng:
```csharp
private SwsContext* Scaler = null;           // FFmpeg scaler context
private AVFilterGraph* FilterGraph = null;   // Filter graph (nếu có)
private AVBufferRef* HardwareDeviceContext = null; // HW device context
```

### 2.2. Quy trình giải mã Video (Step-by-step)

#### Bước 1: Đọc Packet
- **Worker:** `PacketReadingWorker` đọc liên tục packets từ MediaContainer
- **File:** `Unosquare.FFME/Engine/PacketReadingWorker.cs`
- Giữ buffer ~1 giây packets để đảm bảo playback mượt

#### Bước 2: Decode Frame
**File:** `Unosquare.FFME/Engine/FrameDecodingWorker.cs`

```csharp
// FrameDecodingWorker.cs:165
private bool AddNextBlock(MediaType t)
{
    // Decode frames từ packets
    var block = MediaCore.Blocks[t].Add(
        Container.Components[t].ReceiveNextFrame(),
        Container
    );
    return block != null;
}
```

**Luồng decode chi tiết:**

1. **Gọi ReceiveNextFrame()**: Lấy frame tiếp theo từ codec
2. **CreateFrameSource()**: Tạo VideoFrame từ AVFrame* pointer

**File:** `VideoComponent.cs:335-390`

```csharp
protected override MediaFrame CreateFrameSource(IntPtr framePointer)
{
    var frame = (AVFrame*)framePointer;

    // Validate frame
    if (framePointer == IntPtr.Zero || frame->width <= 0 || frame->height <= 0)
        return null;

    // QUAN TRỌNG: Transfer frame từ GPU memory → RAM (nếu dùng HW decode)
    if (HardwareAccelerator != null)
    {
        frame = HardwareAccelerator.ExchangeFrame(CodecContext, frame,
                                                   out var isHardwareFrame);
        IsUsingHardwareDecoding = isHardwareFrame;
    }

    // Apply filter graph (nếu có)
    InitializeFilterGraph(frame);

    if (FilterGraph != null)
    {
        outputFrame = MediaFrame.CloneAVFrame(frame);
        ffmpeg.av_buffersrc_add_frame(SourceFilter, outputFrame);
        ffmpeg.av_buffersink_get_frame_flags(SinkFilter, outputFrame, 0);
    }

    // Tạo VideoFrame wrapper
    return new VideoFrame(outputFrame, this);
}
```

#### Bước 3: Convert Frame → Block (MaterializeFrame)

**File:** `VideoComponent.cs:211-332`

Đây là bước quan trọng nhất - convert frame thô sang format BGRA chuẩn:

```csharp
public override bool MaterializeFrame(MediaFrame input,
                                      ref MediaBlock output,
                                      MediaBlock previousBlock)
{
    var source = input as VideoFrame;
    var target = output as VideoBlock;

    // BƯỚC 1: Tạo hoặc lấy Scaler context
    var newScaler = ffmpeg.sws_getCachedContext(
        Scaler,
        source.Pointer->width,        // Kích thước nguồn
        source.Pointer->height,
        NormalizePixelFormat(source.Pointer), // Format nguồn (YUV, RGB, etc.)
        source.Pointer->width,        // Kích thước đích (giống nguồn)
        source.Pointer->height,
        Constants.VideoPixelFormat,   // AV_PIX_FMT_BGRA (format đích)
        ScalerFlags,                  // SWS_POINT (nearest neighbor)
        null, null, null
    );

    // BƯỚC 2: Cấp phát buffer cho block
    target.Allocate(source, Constants.VideoPixelFormat);

    // BƯỚC 3: Thực hiện scaling/conversion
    if (target.TryAcquireWriterLock(out var writeLock))
    {
        using (writeLock)
        {
            var targetStride = new[] { target.PictureBufferStride };
            var targetScan = default(byte_ptrArray8);
            targetScan[0] = (byte*)target.Buffer;

            // Gọi FFmpeg sws_scale để convert
            var outputHeight = ffmpeg.sws_scale(
                Scaler,
                source.Pointer->data,      // Dữ liệu nguồn
                source.Pointer->linesize,  // Stride nguồn
                0,                         // Start từ line 0
                source.Pointer->height,    // Số lines
                targetScan,                // Buffer đích
                targetStride               // Stride đích
            );
        }
    }

    // BƯỚC 4: Copy timing và metadata
    target.StartTime = source.StartTime;
    target.Duration = source.Duration;
    target.DisplayPictureNumber = source.DisplayPictureNumber;
    target.PictureType = source.PictureType;

    return true;
}
```

**Giải thích chi tiết về Scaler:**

- **sws_getCachedContext()**: Tạo hoặc tái sử dụng scaler context
- **ScalerFlags = SWS_POINT**: Sử dụng nearest neighbor (nhanh nhất, vì không scale kích thước)
- **Pixel Format Conversion**: Chuyển từ YUV/RGB... sang BGRA32 (format WPF yêu cầu)

**Pixel Format hỗ trợ:**
```csharp
// Constants.VideoPixelFormat = AVPixelFormat.AV_PIX_FMT_BGRA
// Output: 32-bit BGRA (B=8bit, G=8bit, R=8bit, A=8bit)
```

#### Bước 4: Buffer Block
- VideoBlock được lưu vào `MediaBlockBuffer`
- BlockRenderingWorker sẽ đọc từ buffer này để render

### 2.3. VideoBlock - Container dữ liệu video

**File:** `Unosquare.FFME/Container/VideoBlock.cs`

```csharp
internal sealed class VideoBlock : MediaBlock
{
    public int PixelWidth { get; private set; }
    public int PixelHeight { get; private set; }
    public int PictureBufferStride { get; private set; }
    public bool IsHardwareFrame { get; internal set; }
    public string HardwareAcceleratorName { get; internal set; }

    // Buffer allocation
    internal unsafe bool Allocate(VideoFrame source, AVPixelFormat pixelFormat)
    {
        // Tính buffer size cần thiết
        var targetLength = ffmpeg.av_image_get_buffer_size(
            pixelFormat,
            source.Pointer->width,
            source.Pointer->height,
            1
        );

        // Cấp phát bộ nhớ unmanaged
        if (!Allocate(targetLength))
            return false;

        // Update properties
        PictureBufferStride = ffmpeg.av_image_get_linesize(
            pixelFormat,
            source.Pointer->width,
            0
        );
        PixelWidth = source.Pointer->width;
        PixelHeight = source.Pointer->height;

        return true;
    }
}
```

**Memory Layout của VideoBlock:**
```
Buffer = unmanaged memory pointer
├─ Stride = width * 4 bytes (BGRA = 4 bytes/pixel)
├─ Size = Stride * Height
└─ Data format: B G R A | B G R A | B G R A | ...
                pixel 1   pixel 2   pixel 3
```

---

## 3. CƠ CHẾ HARDWARE ACCELERATION (GPU DECODING)

### 3.1. HardwareAccelerator Class

**File:** `Unosquare.FFME/Container/HardwareAccelerator.cs`

Đây là thành phần quan trọng nhất cho việc decode trên GPU.

#### Các GPU device types được hỗ trợ:

```csharp
// Từ FFmpeg AVHWDeviceType enum:
- AV_HWDEVICE_TYPE_CUDA       // NVIDIA CUDA
- AV_HWDEVICE_TYPE_DXVA2      // DirectX Video Acceleration 2
- AV_HWDEVICE_TYPE_D3D11VA    // Direct3D 11 Video Acceleration
- AV_HWDEVICE_TYPE_QSV        // Intel Quick Sync Video
- AV_HWDEVICE_TYPE_VAAPI      // Video Acceleration API (Linux)
- AV_HWDEVICE_TYPE_VIDEOTOOLBOX // Apple VideoToolbox (macOS)
```

### 3.2. Khởi tạo Hardware Decoder

**File:** `VideoComponent.cs:164-195`

```csharp
public bool AttachHardwareDevice(HardwareDeviceInfo[] selectedConfigs)
{
    foreach (var selectedConfig in selectedConfigs)
    {
        try
        {
            AVBufferRef* devContextRef = null;

            // BƯỚC 1: Tạo hardware device context
            var initResultCode = ffmpeg.av_hwdevice_ctx_create(
                &devContextRef,              // Output context
                selectedConfig.DeviceType,   // Device type (CUDA, DXVA2, etc.)
                null,                        // Device name (null = default)
                null,                        // Options
                0                            // Flags
            );

            if (initResultCode < 0)
                continue;  // Thử device type khác

            // BƯỚC 2: Tạo HardwareAccelerator object
            var accelerator = new HardwareAccelerator(this, selectedConfig);

            // BƯỚC 3: Gán context cho codec
            HardwareDeviceContext = devContextRef;
            HardwareAccelerator = accelerator;
            CodecContext->hw_device_ctx = ffmpeg.av_buffer_ref(HardwareDeviceContext);

            // BƯỚC 4: Set callback để resolve pixel format
            CodecContext->get_format = accelerator.GetFormatCallback;

            return true;
        }
        catch (Exception ex)
        {
            // Log error và thử device khác
        }
    }

    return false;
}
```

### 3.3. Query Hardware Devices

**File:** `HardwareAccelerator.cs:59-85`

```csharp
public static List<HardwareDeviceInfo> GetCompatibleDevices(AVCodecID codecId)
{
    const int AV_CODEC_HW_CONFIG_METHOD_HW_DEVICE_CTX = 0x01;

    var codec = ffmpeg.avcodec_find_decoder(codecId);
    var result = new List<HardwareDeviceInfo>(64);
    var configIndex = 0;

    // Iterate qua tất cả hardware configs của codec
    while (true)
    {
        var config = ffmpeg.avcodec_get_hw_config(codec, configIndex);
        if (config == null) break;

        // Chỉ lấy configs hỗ trợ hardware device context
        if ((config->methods & AV_CODEC_HW_CONFIG_METHOD_HW_DEVICE_CTX) != 0
            && config->device_type != AVHWDeviceType.AV_HWDEVICE_TYPE_NONE)
        {
            result.Add(new HardwareDeviceInfo(config));
        }

        configIndex++;
    }

    return result;
}
```

**Cách sử dụng:**
```csharp
// Ví dụ: Query devices cho H.264
var devices = HardwareAccelerator.GetCompatibleDevices(AVCodecID.AV_CODEC_ID_H264);
// Kết quả có thể bao gồm: CUDA, DXVA2, D3D11VA, QSV, etc.
```

### 3.4. Transfer Frame từ GPU → CPU

**File:** `HardwareAccelerator.cs:98-123`

Đây là bước QUAN TRỌNG NHẤT khi decode bằng GPU:

```csharp
public AVFrame* ExchangeFrame(AVCodecContext* codecContext,
                              AVFrame* input,
                              out bool isHardwareFrame)
{
    isHardwareFrame = false;

    // Kiểm tra có hardware context không
    if (codecContext->hw_device_ctx == null)
        return input;  // Không có HW, trả về frame gốc

    isHardwareFrame = true;

    // Kiểm tra pixel format có phải HW format không
    if (input->format != (int)PixelFormat)
        return input;  // Không phải HW frame

    // BƯỚC 1: Tạo frame mới trong CPU memory
    var output = MediaFrame.CreateAVFrame();

    // BƯỚC 2: Transfer data từ GPU → CPU
    var result = ffmpeg.av_hwframe_transfer_data(
        output,    // CPU frame (destination)
        input,     // GPU frame (source)
        0          // Flags
    );

    // BƯỚC 3: Copy metadata (timing, etc.)
    ffmpeg.av_frame_copy_props(output, input);

    if (result < 0)
    {
        MediaFrame.ReleaseAVFrame(output);
        throw new MediaContainerException("Failed to transfer data to output frame");
    }

    // BƯỚC 4: Release GPU frame
    MediaFrame.ReleaseAVFrame(input);

    // Trả về CPU frame
    return output;
}
```

**Giải thích chi tiết:**

1. **GPU Decoding**: Frame được decode trên GPU, lưu trong GPU memory
2. **av_hwframe_transfer_data()**: Copy frame từ VRAM (GPU) → RAM (CPU)
3. **Tại sao phải transfer?**: WPF renderer cần data ở CPU memory để vẽ lên bitmap

**Memory Flow:**
```
Video Packet → GPU Decoder → GPU Memory (VRAM)
                                  ↓
                    av_hwframe_transfer_data()
                                  ↓
                            CPU Memory (RAM)
                                  ↓
                              Scaler
                                  ↓
                          BGRA Block Buffer
                                  ↓
                         WPF WriteableBitmap
```

### 3.5. Pixel Format Callback

**File:** `HardwareAccelerator.cs:132-153`

```csharp
private AVPixelFormat GetPixelFormat(AVCodecContext* context,
                                     AVPixelFormat* pixelFormats)
{
    // Default: format đầu tiên
    var output = *pixelFormats;

    // Iterate qua các pixel formats codec hỗ trợ
    for (var p = pixelFormats; *p != AVPixelFormat.AV_PIX_FMT_NONE; p++)
    {
        // Tìm format khớp với HW device
        if (*p == PixelFormat)
        {
            output = PixelFormat;  // Dùng HW format
            break;
        }

        // Fallback: SW format
        output = *p;
    }

    return output;
}
```

**Ví dụ Hardware Pixel Formats:**
- CUDA: `AV_PIX_FMT_CUDA`
- DXVA2: `AV_PIX_FMT_DXVA2_VLD`
- D3D11: `AV_PIX_FMT_D3D11`
- QSV: `AV_PIX_FMT_QSV`

### 3.6. HardwareDeviceInfo

**File:** `Unosquare.FFME/Common/HardwareDeviceInfo.cs`

```csharp
public sealed unsafe class HardwareDeviceInfo
{
    public AVHWDeviceType DeviceType { get; }        // Loại device
    public string DeviceTypeName { get; }            // Tên (CUDA, DXVA2, etc.)
    public AVPixelFormat PixelFormat { get; }        // Pixel format
    public string PixelFormatName { get; }           // Tên format

    internal HardwareDeviceInfo(AVCodecHWConfig* config)
    {
        DeviceType = config->device_type;
        PixelFormat = config->pix_fmt;
        DeviceTypeName = ffmpeg.av_hwdevice_get_type_name(DeviceType);
        PixelFormatName = ffmpeg.av_get_pix_fmt_name(PixelFormat);
    }
}
```

---

## 4. RENDERING - PHÁT VIDEO LÊN SCREEN

### 4.1. BlockRenderingWorker

**File:** `Unosquare.FFME/Engine/BlockRenderingWorker.cs`

Worker chạy với priority cao để đảm bảo rendering mượt.

```csharp
private void RenderBlock(MediaType t)
{
    var playbackClock = MediaCore.Timing.GetPosition(t);

    // Lấy block tại vị trí clock hiện tại
    var currentBlock = MediaCore.Blocks[t][playbackClock.Ticks];

    // Gửi block đến renderer
    SendBlockToRenderer(currentBlock, playbackClock);

    // Update renderer với clock mới
    MediaCore.Renderers[t]?.Update(playbackClock);
}
```

**Timing và Synchronization:**
- Sử dụng Real-Time Clock (RTC) để đồng bộ A/V
- Hỗ trợ V-Sync cho video rendering mượt
- VerticalSyncContext để sync với màn hình (60Hz, 120Hz, etc.)

### 4.2. VideoRenderer - WPF WriteableBitmap

**File:** `Unosquare.FFME.Windows/Rendering/VideoRenderer.cs`

Đây là renderer mặc định sử dụng WriteableBitmap của WPF.

```csharp
public override void Render(MediaBlock mediaBlock, TimeSpan clockPosition)
{
    var block = BeginRenderingCycle(mediaBlock);
    if (block == null) return;

    VideoDispatcher?.Invoke(() =>
    {
        try
        {
            // Prepare bitmap
            if (PrepareVideoFrameBuffer(block))
                WriteVideoFrameBuffer(block, clockPosition);
        }
        finally
        {
            FinishRenderingCycle(block, clockPosition);
        }
    }, DispatcherPriority.Normal);
}
```

#### PrepareVideoFrameBuffer:

```csharp
private bool PrepareVideoFrameBuffer(VideoBlock block)
{
    // Kiểm tra cần tạo bitmap mới không
    var needsCreation = (TargetBitmap == null) && MediaElement.HasVideo;

    var needsModification = TargetBitmap != null &&
        (TargetBitmapData.PixelWidth != block.PixelWidth ||
         TargetBitmapData.PixelHeight != block.PixelHeight ||
         TargetBitmapData.Stride != block.PictureBufferStride);

    if (needsCreation || needsModification)
    {
        // Tạo WriteableBitmap mới
        TargetBitmap = new WriteableBitmap(
            block.PixelWidth,
            block.PixelHeight,
            DpiX, DpiY,
            PixelFormats.Bgra32,  // BGRA format
            null
        );
    }

    return TargetBitmapData != null;
}
```

#### WriteVideoFrameBuffer - Copy data:

```csharp
private unsafe void WriteVideoFrameBuffer(VideoBlock block, TimeSpan clockPosition)
{
    var bitmap = TargetBitmap;
    var target = TargetBitmapData;

    if (!block.TryAcquireReaderLock(out var readLock))
        return;

    try
    {
        // BƯỚC 1: Lock bitmap để access back buffer
        bitmap.Lock();

        // BƯỚC 2: Tính số bytes cần copy
        var bufferLength = Math.Min(block.BufferLength, target.BufferLength);

        // BƯỚC 3: Copy dữ liệu từ block → bitmap back buffer
        Buffer.MemoryCopy(
            block.Buffer.ToPointer(),    // Source: VideoBlock buffer
            target.Scan0.ToPointer(),    // Dest: Bitmap back buffer
            bufferLength,
            bufferLength
        );

        // BƯỚC 4: Raise event cho user customization
        MediaElement?.RaiseRenderingVideoEvent(block, target, clockPosition);

        // BƯỚC 5: Mark dirty rect để WPF update UI
        bitmap.AddDirtyRect(target.UpdateRect);
    }
    finally
    {
        readLock.Dispose();
        bitmap.Unlock();
    }
}
```

**Memory Copy Performance:**
- Sử dụng `Buffer.MemoryCopy()` - hàm unsafe cực nhanh
- Copy trực tiếp vào back buffer của bitmap
- Không có allocation hay copying thừa

### 4.3. InteropVideoRenderer - Memory Mapped Files

**File:** `Unosquare.FFME.Windows/Rendering/InteropVideoRenderer.cs`

Renderer thay thế sử dụng InteropBitmap với memory-mapped files - hiệu quả hơn cho video độ phân giải cao.

```csharp
public unsafe BitmapDataBuffer Write(VideoBlock block)
{
    lock (SyncLock)
    {
        EnsureBuffers(block);

        // Copy vào memory-mapped file
        var bufferLength = Math.Min(block.BufferLength, BackBufferView.Capacity);
        var scan0 = BackBufferView.SafeMemoryMappedViewHandle.DangerousGetHandle();

        Buffer.MemoryCopy(
            block.Buffer.ToPointer(),
            scan0.ToPointer(),
            bufferLength,
            bufferLength
        );

        // Flush để đảm bảo data được write
        BackBufferView.Flush();

        return BitmapData;
    }
}

public void Render(ImageHost host)
{
    lock (SyncLock)
    {
        if (NeedsNewImage)
        {
            // Tạo InteropBitmap từ memory section
            BackBufferImage = (InteropBitmap)Imaging.CreateBitmapSourceFromMemorySection(
                BackBufferFile.SafeMemoryMappedFileHandle.DangerousGetHandle(),
                Width, Height,
                PixelFormats.Bgra32,
                Stride,
                0
            );

            NeedsNewImage = false;
        }

        // Invalidate để force refresh
        BackBufferImage.Invalidate();
        host.Source = BackBufferImage;
    }
}
```

**Ưu điểm của InteropBitmap:**
- Zero-copy rendering
- Shared memory giữa managed và unmanaged code
- Hiệu quả hơn với video 4K, 8K

### 4.4. So sánh 2 Renderers

| Đặc điểm | VideoRenderer | InteropVideoRenderer |
|----------|--------------|---------------------|
| Loại Bitmap | WriteableBitmap | InteropBitmap |
| Memory | Managed heap | Memory-mapped file |
| Performance | Tốt cho HD | Tốt hơn cho 4K+ |
| Complexity | Đơn giản | Phức tạp hơn |
| Copy | Buffer.MemoryCopy | Zero-copy shared mem |

---

## 5. LUỒNG XỬ LÝ HOÀN CHỈNH

### 5.1. Decode trên CPU (Software Decoding)

```
1. PacketReadingWorker
   └─> Read packet from input stream
   └─> Cache packet in MediaComponent

2. FrameDecodingWorker
   └─> Get packet from cache
   └─> Send to FFmpeg decoder (CPU)
   └─> Receive AVFrame (YUV420/RGB/etc.)
   └─> Apply filter graph (optional)
   └─> CreateFrameSource() → VideoFrame

3. MaterializeFrame (Convert)
   └─> Create SwsContext (scaler)
   └─> Allocate VideoBlock buffer
   └─> sws_scale: Convert YUV → BGRA32
   └─> Copy timing metadata
   └─> Store in MediaBlockBuffer

4. BlockRenderingWorker
   └─> Get VideoBlock at current clock position
   └─> Send to VideoRenderer

5. VideoRenderer
   └─> Prepare WriteableBitmap
   └─> Lock bitmap
   └─> Buffer.MemoryCopy: Block → Bitmap
   └─> AddDirtyRect
   └─> Unlock bitmap
   └─> WPF displays on screen
```

### 5.2. Decode trên GPU (Hardware Decoding)

```
1. PacketReadingWorker
   └─> Read packet from input stream
   └─> Cache packet in MediaComponent

2. Khởi tạo Hardware Decoder (1 lần)
   └─> Query compatible devices
   └─> av_hwdevice_ctx_create(CUDA/DXVA2/D3D11/QSV)
   └─> Set CodecContext->hw_device_ctx
   └─> Set get_format callback

3. FrameDecodingWorker
   └─> Get packet from cache
   └─> Send to FFmpeg decoder
   └─> Decoder sử dụng GPU để decode
   └─> Receive AVFrame (in GPU memory - VRAM)

4. CreateFrameSource (trong VideoComponent)
   └─> Check if HardwareAccelerator != null
   └─> Call HardwareAccelerator.ExchangeFrame()
       └─> Create new CPU frame
       └─> av_hwframe_transfer_data(GPU → CPU)
           ├─ Copy từ VRAM → RAM
           └─ This is the GPU → CPU transfer!
       └─> av_frame_copy_props (copy metadata)
       └─> Release GPU frame
       └─> Return CPU frame
   └─> Apply filter graph (optional)
   └─> CreateFrameSource() → VideoFrame

5. MaterializeFrame (Convert)
   └─> Create SwsContext (scaler)
   └─> Allocate VideoBlock buffer
   └─> sws_scale: Convert format → BGRA32
   └─> Store in MediaBlockBuffer

6. BlockRenderingWorker + VideoRenderer
   └─> (Giống như software decoding)
```

**Điểm khác biệt chính:**
- **GPU Decode**: Decode diễn ra trên GPU (CUDA/DXVA2/etc.)
- **Transfer**: `av_hwframe_transfer_data()` copy frame từ VRAM → RAM
- **Performance**: GPU decode nhanh hơn nhiều, đặc biệt với H.264/HEVC 4K+

### 5.3. Parallel vs Serial Processing

**Parallel Decoding:**
```csharp
// FrameDecodingWorker - Parallel mode
ParallelDecodeBlocks = (all, ct) =>
{
    Parallel.ForEach(all, (t) =>
        Interlocked.Add(ref DecodedFrameCount,
        DecodeComponentBlocks(t, ct)));
};
```

**Parallel Rendering:**
```csharp
// BlockRenderingWorker - Parallel mode
ParallelRenderBlocks = (all) =>
    Parallel.ForEach(all, (t) => RenderBlock(t));
```

**Khi nào dùng Parallel:**
- Multiple media types (video + audio + subtitle)
- Disconnected clocks (mỗi component có clock riêng)
- High-performance systems

---

## 6. FFMPEG INTEROP VÀ MEMORY MANAGEMENT

### 6.1. FFmpeg.AutoGen

Dự án sử dụng **FFmpeg.AutoGen v7.0.0** để tạo P/Invoke bindings tự động cho FFmpeg C API.

```csharp
using FFmpeg.AutoGen;

// Ví dụ các hàm FFmpeg được sử dụng:
ffmpeg.sws_scale(...)                     // Scaling/conversion
ffmpeg.avcodec_find_decoder(...)          // Find codec
ffmpeg.avcodec_get_hw_config(...)         // Get HW configs
ffmpeg.av_hwdevice_ctx_create(...)        // Create HW context
ffmpeg.av_hwframe_transfer_data(...)      // Transfer GPU → CPU
ffmpeg.av_image_get_buffer_size(...)      // Calculate buffer size
ffmpeg.avfilter_graph_create_filter(...)  // Create filter
```

### 6.2. Unsafe Code và Pointers

Code sử dụng rộng rãi `unsafe` để làm việc với unmanaged memory:

```csharp
unsafe class VideoComponent : MediaComponent
{
    private SwsContext* Scaler = null;
    private AVFilterGraph* FilterGraph = null;

    protected override MediaFrame CreateFrameSource(IntPtr framePointer)
    {
        var frame = (AVFrame*)framePointer;
        // ...
    }
}
```

### 6.3. Resource Management

**RC (Reference Counter):**
```csharp
// Thêm resource để track
RC.Current.Add(Scaler);
RC.Current.Add(FilterGraph);

// Remove khi không dùng nữa
RC.Current.Remove(Scaler);
ffmpeg.sws_freeContext(Scaler);
```

**MediaFrame Management:**
```csharp
// Create AVFrame
var outputFrame = MediaFrame.CreateAVFrame();

// Clone AVFrame
var cloned = MediaFrame.CloneAVFrame(frame);

// Release AVFrame
MediaFrame.ReleaseAVFrame(frame);
```

**VideoBlock Buffer:**
- Allocate bằng unmanaged memory
- Sử dụng lock để thread-safe
- Deallocate khi dispose

---

## 7. CÁC TÍNH NĂNG NÂNG CAO

### 7.1. FFmpeg Filter Graph

**File:** `VideoComponent.cs:520-647`

Hỗ trợ áp dụng filter graphs của FFmpeg (deinterlace, denoise, etc.):

```csharp
private void InitializeFilterGraph(AVFrame* frame)
{
    // Tạo filter graph
    FilterGraph = ffmpeg.avfilter_graph_alloc();

    // Create source filter (buffer)
    ffmpeg.avfilter_graph_create_filter(
        &sourceFilterRef,
        ffmpeg.avfilter_get_by_name("buffer"),
        "video_buffer",
        filterArguments,
        null,
        FilterGraph
    );

    // Create sink filter (buffersink)
    ffmpeg.avfilter_graph_create_filter(
        &sinkFilterRef,
        ffmpeg.avfilter_get_by_name("buffersink"),
        "video_buffersink",
        null, null, FilterGraph
    );

    // Parse filter string (ví dụ: "yadif=0:-1:0" cho deinterlace)
    ffmpeg.avfilter_graph_parse(FilterGraph, filterString,
                                 SinkInput, SourceOutput, null);

    // Configure graph
    ffmpeg.avfilter_graph_config(FilterGraph, null);
}
```

**Cách sử dụng:**
```csharp
Container.MediaOptions.VideoFilter = "yadif=0:-1:0";  // Deinterlace
Container.MediaOptions.VideoFilter = "hqdn3d";         // Denoise
```

### 7.2. Closed Captions (CEA-608/708)

**File:** `VideoFrame.cs` và `VideoBlock.cs`

Hỗ trợ extract và render closed captions:

```csharp
public IReadOnlyList<ClosedCaptionPacket> ClosedCaptions { get; internal set; }
```

Extract từ side data của AVFrame.

### 7.3. Seek và Frame-by-Frame

**Video Seek Index:**
```csharp
public IList<VideoSeekIndexEntry> SeekIndex { get; }
```

Hỗ trợ:
- Fast seeking
- Frame-by-frame stepping (left/right arrow)
- Precise seeking với seek index

### 7.4. Live Streaming

Hỗ trợ các protocols:
- HLS (HTTP Live Streaming)
- RTSP
- RTMP
- HTTP/HTTPS

**Buffer management cho live streams:**
```csharp
private void CatchUpWithLiveStream()
{
    // Adjust speed ratio để catch up với live stream
    // Hoặc adjust timing để giảm lag
}
```

### 7.5. Capture Devices

Hỗ trợ capture từ:
- Webcam
- Screen capture (GDI grab)
- Audio devices

```
device://dshow/?audio=Microphone:video=Webcam
device://gdigrab?desktop
```

---

## 8. PERFORMANCE VÀ OPTIMIZATION

### 8.1. Multi-threading Architecture

**3 Worker Threads:**
1. **PacketReadingWorker**: Normal priority
2. **FrameDecodingWorker**: Above normal priority
3. **BlockRenderingWorker**: Highest priority

**Thread Communication:**
- Lock-free queues khi có thể
- Reader/Writer locks cho shared buffers
- Atomic operations (AtomicBoolean, Interlocked)

### 8.2. Memory Optimization

**Buffer Pooling:**
- MediaBlockBuffer reuse blocks
- SwsContext caching (sws_getCachedContext)
- AVFrame pooling

**Zero-Copy Techniques:**
- InteropBitmap với memory-mapped files
- Direct pointer access với unsafe code
- Buffer.MemoryCopy thay vì managed array copy

### 8.3. V-Sync Support

**File:** `BlockRenderingWorker.cs:187-227`

```csharp
using var vsync = new VerticalSyncContext();

while (WorkerState != WorkerState.Stopped)
{
    if (State.VerticalSyncEnabled && Container.Components.HasVideo)
    {
        // Wait for vertical blank
        while (RemainingCycleTime.Ticks >= vsync.RefreshPeriod.Ticks * 2)
            vsync.WaitForBlank();

        // Final sync
        vsync.WaitForBlank();
    }
    else
    {
        // Synthetic wait
        QuantumWaiter.Wait(RemainingCycleTime);
    }

    ExecuteCycle();
}
```

**Lợi ích:**
- Giảm screen tearing
- Smooth playback
- Sync với monitor refresh rate (60/120/144Hz)

### 8.4. Benchmarks (Ước tính)

**Software Decode (CPU):**
- H.264 1080p: ~30-50% CPU (quad-core)
- HEVC 4K: ~80-100% CPU

**Hardware Decode (GPU):**
- H.264 1080p: ~5-10% CPU, ~15-20% GPU
- HEVC 4K: ~10-15% CPU, ~25-35% GPU

**Rendering:**
- WriteableBitmap: ~5-10ms per frame (1080p)
- InteropBitmap: ~3-5ms per frame (1080p)

---

## 9. CÁC ĐIỂM QUAN TRỌNG CẦN NHỚ

### 9.1. Về Video Decoding

1. **Software Decode:**
   - Decoder chạy trên CPU
   - Output là AVFrame trong RAM
   - Không cần transfer GPU → CPU

2. **Hardware Decode:**
   - Decoder chạy trên GPU (CUDA/DXVA2/D3D11/QSV)
   - Output là AVFrame trong VRAM (GPU memory)
   - **BẮT BUỘC** phải transfer từ GPU → CPU bằng `av_hwframe_transfer_data()`
   - Transfer này là overhead nhưng vẫn nhanh hơn software decode nhiều

3. **Pixel Format Conversion:**
   - FFmpeg decoder output: YUV420P, YUV422P, RGB, etc. (codec-dependent)
   - WPF yêu cầu: BGRA32 (32-bit, 8-bit per component)
   - Sử dụng `sws_scale()` để convert
   - ScalerFlags = SWS_POINT (nearest neighbor) vì không scale size

### 9.2. Về GPU Rendering

**Lưu ý:** FFME không render video trực tiếp trên GPU. Nó render trên CPU thông qua WPF.

**Luồng data:**
```
GPU Decode → VRAM (GPU) → av_hwframe_transfer_data() → RAM (CPU)
                                                           ↓
                                                    sws_scale()
                                                           ↓
                                                    BGRA Buffer
                                                           ↓
                                               WriteableBitmap (CPU)
                                                           ↓
                                                   WPF Rendering
                                                           ↓
                                              GPU (DirectX/WPF)
```

**WPF tự động render bitmap lên GPU**, nhưng video processing (decode → convert → copy) chủ yếu trên CPU.

### 9.3. Về Multi-GPU

FFME **KHÔNG HỖ TRỢ** chọn GPU cụ thể. Nó dùng GPU default của system:

```csharp
// av_hwdevice_ctx_create với device name = null → dùng default GPU
ffmpeg.av_hwdevice_ctx_create(&devContextRef, selectedConfig.DeviceType,
                               null, // ← null = default device
                               null, 0);
```

**Để chọn GPU cụ thể:**
- CUDA: Cần set CUDA device ID qua options
- D3D11: Cần pass ID3D11Device* qua options
- Yêu cầu modify code

### 9.4. Về Performance

**Hardware Decode WIN khi:**
- Video resolution cao (4K, 8K)
- H.264/H.265/VP9 (codecs có HW support tốt)
- Laptop/low-power devices
- Muốn giảm CPU usage

**Software Decode WIN khi:**
- Codecs không có HW support (AV1 trên GPU cũ)
- Cần áp dụng nhiều filters
- GPU bận xử lý khác (gaming, etc.)

---

## 10. CODE EXAMPLES

### 10.1. Khởi tạo và mở media với HW decode

```csharp
// Set FFmpeg directory
Unosquare.FFME.Library.FFmpegDirectory = @"c:\ffmpeg";

// Create MediaElement
var mediaElement = new MediaElement();

// Setup HW decode cho H.264
var container = mediaElement.MediaCore.Container;
var videoComponent = container.Components.Video;

// Query available HW decoders
var hwDevices = HardwareAccelerator.GetCompatibleDevices(
    FFmpeg.AutoGen.AVCodecID.AV_CODEC_ID_H264
);

if (hwDevices.Count > 0)
{
    // Try attach HW decoder
    // Priority: CUDA > D3D11VA > DXVA2 > QSV
    var attached = videoComponent.AttachHardwareDevice(hwDevices.ToArray());

    if (attached)
        Console.WriteLine($"Hardware decoder attached: {hwDevices[0].DeviceTypeName}");
    else
        Console.WriteLine("Failed to attach HW decoder, using software decode");
}

// Open media
await mediaElement.Open(new Uri("https://example.com/video.mp4"));

// Play
await mediaElement.Play();
```

### 10.2. Custom rendering - Modify video frames

```csharp
mediaElement.RenderingVideo += (s, e) =>
{
    var block = e.EngineBlock as VideoBlock;
    var bitmap = e.Bitmap;

    // Access raw BGRA data
    unsafe
    {
        var scan0 = (byte*)bitmap.Scan0.ToPointer();
        var stride = bitmap.Stride;
        var width = bitmap.PixelWidth;
        var height = bitmap.PixelHeight;

        // Example: Apply grayscale effect
        for (int y = 0; y < height; y++)
        {
            for (int x = 0; x < width; x++)
            {
                var offset = y * stride + x * 4;
                var b = scan0[offset + 0];
                var g = scan0[offset + 1];
                var r = scan0[offset + 2];

                // Grayscale = 0.299*R + 0.587*G + 0.114*B
                var gray = (byte)(0.299 * r + 0.587 * g + 0.114 * b);

                scan0[offset + 0] = gray;  // B
                scan0[offset + 1] = gray;  // G
                scan0[offset + 2] = gray;  // R
                // scan0[offset + 3] = alpha (không đổi)
            }
        }
    }
};
```

### 10.3. Apply FFmpeg video filter

```csharp
// Before opening media
container.MediaOptions.VideoFilter = "yadif=0:-1:0";  // Deinterlace

// Or change dynamically (requires ChangeMedia command)
await mediaElement.ChangeMedia();
```

### 10.4. Frame-by-frame seeking

```csharp
// Seek 1 frame forward
await mediaElement.StepForward();

// Seek 1 frame backward
await mediaElement.StepBackward();

// Seek to specific position
await mediaElement.Seek(TimeSpan.FromSeconds(10));
```

---

## 11. KẾT LUẬN

### 11.1. Điểm mạnh của FFMediaElement

1. **Hỗ trợ đầy đủ codecs** nhờ FFmpeg (H.264/265, VP9, AV1, etc.)
2. **Hardware acceleration** cho decode (CUDA, DXVA2, D3D11, QSV)
3. **Multi-threaded architecture** tối ưu performance
4. **Frame-level control** (seeking, filtering, custom rendering)
5. **Live streaming support** (HLS, RTSP, RTMP)
6. **Production-ready** với error handling và logging tốt

### 11.2. Điểm cần cải thiện

1. **GPU Rendering**: Hiện tại render qua CPU (WPF bitmap)
   - Có thể cải thiện bằng Direct3D/SharpDX rendering

2. **Multi-GPU Support**: Không hỗ trợ chọn GPU cụ thể
   - Cần thêm options để specify GPU device

3. **Memory Usage**: Có thể optimize thêm với larger buffer pools

4. **Zero-copy từ GPU**: Hiện phải transfer GPU → CPU
   - Lý tưởng: Decode trên GPU, render trực tiếp trên GPU không qua CPU

### 11.3. Ứng dụng thực tế

FFME phù hợp cho:
- **Media players** (như VLC, MPC-HC)
- **Video editors** (timeline playback)
- **Streaming clients** (HLS, RTSP viewers)
- **Video surveillance systems**
- **Digital signage**
- **Medical imaging** (DICOM video)

### 11.4. Tài liệu tham khảo

- **FFmpeg Documentation**: https://ffmpeg.org/documentation.html
- **FFmpeg Hardware Acceleration**: https://trac.ffmpeg.org/wiki/HWAccelIntro
- **WPF WriteableBitmap**: https://docs.microsoft.com/en-us/dotnet/api/system.windows.media.imaging.writeablebitmap
- **FFME GitHub**: https://github.com/unosquare/ffmediaelement

---

## PHỤ LỤC: CẤU TRÚC THƯ MỤC DỰ ÁN

```
ffmediaelement/
├── Unosquare.FFME/                          # Core library (shared project)
│   ├── Container/
│   │   ├── MediaComponent.cs                # Base component class
│   │   ├── VideoComponent.cs                # Video decoder component ⭐
│   │   ├── AudioComponent.cs                # Audio decoder component
│   │   ├── SubtitleComponent.cs             # Subtitle component
│   │   ├── VideoFrame.cs                    # AVFrame wrapper
│   │   ├── VideoBlock.cs                    # Converted video block ⭐
│   │   ├── HardwareAccelerator.cs           # GPU decode support ⭐
│   │   └── MediaContainer.cs                # Input stream wrapper
│   ├── Engine/
│   │   ├── MediaEngine.cs                   # Core engine
│   │   ├── PacketReadingWorker.cs           # Packet reader thread
│   │   ├── FrameDecodingWorker.cs           # Frame decoder thread ⭐
│   │   ├── BlockRenderingWorker.cs          # Block renderer thread ⭐
│   │   └── TimingController.cs              # A/V sync
│   ├── Common/
│   │   ├── HardwareDeviceInfo.cs            # HW device info ⭐
│   │   └── MediaBlockBuffer.cs              # Block buffer
│   └── FFmpeg/
│       └── FFInterop.cs                     # FFmpeg interop helpers
│
├── Unosquare.FFME.Windows/                  # Windows/WPF implementation
│   ├── Rendering/
│   │   ├── VideoRenderer.cs                 # WriteableBitmap renderer ⭐
│   │   ├── InteropVideoRenderer.cs          # InteropBitmap renderer ⭐
│   │   ├── AudioRenderer.cs                 # Audio renderer
│   │   └── SubtitleRenderer.cs              # Subtitle renderer
│   ├── Platform/
│   │   └── VerticalSyncContext.cs           # V-Sync support
│   └── MediaElement.cs                      # Main WPF control
│
└── Unosquare.FFME.Windows.Sample/           # Sample application
    └── ffmeplay (demo player)

⭐ = Files quan trọng nhất cho video decode và GPU rendering
```

---

**HẾT PHÂN TÍCH**

*File này cung cấp phân tích chi tiết về cơ chế decode video và GPU rendering trong ffmediaelement. Để biết thêm chi tiết, vui lòng tham khảo source code tại các file được đánh dấu ⭐ ở trên.*
