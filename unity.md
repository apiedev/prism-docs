---
title: Unity Integration
---

# Prism Unity Plugin

High-performance native video player for Unity with HLS/DASH streaming and platform URL resolution.

## Overview

The Prism Unity Plugin provides native video playback capabilities for Unity applications, supporting:

- **Native Decoders** - Windows Media Foundation, AVFoundation (macOS), GStreamer (Linux)
- **FFmpeg Plugin** - HLS, DASH, RTMP, RTSP streaming with broad codec support
- **URL Resolution** - Resolve YouTube, Twitch, Vimeo URLs to direct streams

## Requirements

- Unity 2020.3 or later
- 64-bit builds only
- Windows 10+ (macOS and Linux support planned)

## Installation

### Via Unity Package Manager

1. Open **Window > Package Manager**
2. Click **+** > **Add package from git URL**
3. Enter:
   ```
   https://github.com/apiedev/prism-unity-plugin.git
   ```

### Manual Installation

1. Clone or download the repository
2. Copy to your project's `Packages` folder
3. Or import as a local package via Package Manager

## Setup

### Step 1: Download FFmpeg Libraries

The FFmpeg DLLs are not included in git due to size constraints (~200MB). Run the download script:

**Windows (PowerShell):**
```powershell
cd Packages/prism-unity-plugin
.\Scripts\download_ffmpeg.ps1
```

**Linux/macOS:**
```bash
cd Packages/prism-unity-plugin
chmod +x Scripts/download_ffmpeg.sh
./Scripts/download_ffmpeg.sh
```

### Step 2: Verify Plugin Files

After setup, check `Plugins/Windows/x86_64/` contains:

| File | Size | Description |
|------|------|-------------|
| `prism_native_win.dll` | ~30 KB | Windows Media Foundation player |
| `prism_ffmpeg.dll` | ~30 KB | FFmpeg decoder plugin |
| `prism_ytdlp.dll` | ~25 KB | yt-dlp URL resolver |
| `avcodec-62.dll` | ~100 MB | FFmpeg codec library |
| `avformat-62.dll` | ~22 MB | FFmpeg format library |
| `avutil-60.dll` | ~3 MB | FFmpeg utility library |
| `swscale-9.dll` | ~2 MB | FFmpeg scaling library |
| `swresample-6.dll` | ~700 KB | FFmpeg resampling library |

## Quick Start

### Basic Video Playback

```csharp
using PrismVideo;
using UnityEngine;
using UnityEngine.UI;

public class SimpleVideoPlayer : MonoBehaviour
{
    public RawImage display;
    public string videoUrl = "https://example.com/video.mp4";

    private IPrismPlayer player;

    void Start()
    {
        // Create native player (no FFmpeg required for MP4)
        player = PrismPlayerFactory.CreateNativePlayer();
        player.Open(videoUrl);
        player.Play();
    }

    void Update()
    {
        player?.Update(Time.deltaTime);

        if (player?.VideoTexture != null)
        {
            display.texture = player.VideoTexture;
        }
    }

    void OnDestroy()
    {
        player?.Dispose();
    }
}
```

### HLS/DASH Streaming

For streaming protocols, use the FFmpeg player:

```csharp
using PrismVideo;
using UnityEngine;

public class HLSPlayer : MonoBehaviour
{
    public string hlsUrl = "https://example.com/stream.m3u8";

    private PrismFFmpegPlayer player;

    void Start()
    {
        player = gameObject.AddComponent<PrismFFmpegPlayer>();
        player.Open(hlsUrl);
        player.Play();
    }
}
```

### YouTube/Twitch Playback

Resolve platform URLs to direct streams using yt-dlp:

```csharp
using PrismVideo;
using PrismVideo.Streaming;
using UnityEngine;

public class TwitchStreamPlayer : MonoBehaviour
{
    public string channelName = "shroud";

    private PrismFFmpegPlayer player;

    async void Start()
    {
        player = gameObject.AddComponent<PrismFFmpegPlayer>();

        // Resolve Twitch URL to HLS stream
        var resolver = new YtdlpResolver();
        var result = await resolver.ResolveAsync($"https://twitch.tv/{channelName}");

        if (result.Success)
        {
            Debug.Log($"Stream URL: {result.StreamUrl}");
            Debug.Log($"Title: {result.Title}");
            Debug.Log($"Is Live: {result.IsLive}");

            player.Open(result.StreamUrl);
            player.Play();
        }
        else
        {
            Debug.LogError($"Failed to resolve: {result.Error}");
        }
    }
}
```

### YouTube Video Playback

```csharp
using PrismVideo;
using PrismVideo.Streaming;
using UnityEngine;

public class YouTubePlayer : MonoBehaviour
{
    public string videoUrl = "https://www.youtube.com/watch?v=dQw4w9WgXcQ";
    public StreamQuality quality = StreamQuality.Quality720p;

    private PrismFFmpegPlayer player;

    async void Start()
    {
        player = gameObject.AddComponent<PrismFFmpegPlayer>();

        var resolver = new YtdlpResolver();
        var result = await resolver.ResolveAsync(videoUrl, quality);

        if (result.Success)
        {
            player.Open(result.StreamUrl);
            player.Play();
        }
    }
}
```

## API Reference

### IPrismPlayer Interface

The core player interface implemented by all player types:

```csharp
public interface IPrismPlayer : IDisposable
{
    // Lifecycle
    void Open(string url);
    void Play();
    void Pause();
    void Stop();
    void Seek(double positionSeconds);
    void Update(float deltaTime);

    // State
    PrismPlayerState State { get; }
    string ErrorMessage { get; }

    // Time
    double Position { get; }
    double Duration { get; }
    bool IsLive { get; }

    // Video
    Texture2D VideoTexture { get; }
    int VideoWidth { get; }
    int VideoHeight { get; }

    // Audio
    float Volume { get; set; }
    bool IsMuted { get; set; }

    // Playback
    float Speed { get; set; }
    bool Loop { get; set; }
}
```

### PrismPlayerState

```csharp
public enum PrismPlayerState
{
    Idle,       // Initial state, no media loaded
    Opening,    // Loading/buffering media
    Playing,    // Active playback
    Paused,     // Playback paused
    Stopped,    // Playback stopped
    EndOfFile,  // Reached end of media
    Error       // Error occurred, check ErrorMessage
}
```

### StreamQuality

```csharp
public enum StreamQuality
{
    Auto,          // Let yt-dlp choose best quality
    Quality360p,
    Quality480p,
    Quality720p,
    Quality1080p,
    Quality1440p,
    Quality4K,
    AudioOnly
}
```

### IStreamResolver

```csharp
public interface IStreamResolver
{
    bool CanResolve(string url);
    Task<StreamResolveResult> ResolveAsync(string url, StreamQuality quality = StreamQuality.Auto);
}

public class StreamResolveResult
{
    public bool Success { get; }
    public string StreamUrl { get; }
    public string Title { get; }
    public bool IsLive { get; }
    public double Duration { get; }
    public string Error { get; }
}
```

## Player Types

### PrismNativePlayer

Uses the OS native media framework. Best for simple video files.

**Pros:**
- No additional dependencies
- Hardware acceleration built-in
- Low memory footprint

**Cons:**
- Limited format support
- No HLS/DASH on Windows

**Supported formats:** MP4, MOV, WMV, AVI, MP3, WMA

### PrismFFmpegPlayer

Uses the FFmpeg plugin for broad format and protocol support.

**Pros:**
- HLS, DASH, RTMP, RTSP support
- Wide codec compatibility
- Consistent behavior across platforms

**Cons:**
- Requires FFmpeg DLLs (~200MB)
- Higher memory usage

**Supported formats:** Everything FFmpeg supports

## URL Resolution

The yt-dlp resolver supports 1000+ platforms including:

| Platform | Live | VOD | Notes |
|----------|------|-----|-------|
| YouTube | Yes | Yes | Includes Shorts, playlists |
| Twitch | Yes | Yes | Clips, VODs, live streams |
| Vimeo | - | Yes | May require login for some videos |
| Dailymotion | - | Yes | |
| Twitter/X | - | Yes | Video tweets |
| TikTok | Yes | Yes | |
| Instagram | - | Yes | Reels, posts |
| Facebook | Yes | Yes | |

### Resolution Options

```csharp
var resolver = new YtdlpResolver();

// Default quality (best available)
var result = await resolver.ResolveAsync("https://youtube.com/watch?v=...");

// Specific quality
var result = await resolver.ResolveAsync(url, StreamQuality.Quality720p);

// Audio only
var result = await resolver.ResolveAsync(url, StreamQuality.AudioOnly);
```

### URL Expiration

Resolved URLs are temporary and expire. For long sessions, implement refresh logic:

```csharp
private async void RefreshStreamUrl()
{
    while (isPlaying)
    {
        await Task.Delay(TimeSpan.FromMinutes(30));

        var result = await resolver.ResolveAsync(originalUrl);
        if (result.Success)
        {
            // Store new URL for reconnection if needed
            currentStreamUrl = result.StreamUrl;
        }
    }
}
```

## Troubleshooting

### DllNotFoundException: prism_ffmpeg

**Cause:** FFmpeg libraries not installed.

**Solution:**
```powershell
.\Scripts\download_ffmpeg.ps1
```

### Unable to resolve URL / yt-dlp not found

**Cause:** yt-dlp binary not available.

**Solution:** The yt-dlp resolver auto-downloads the binary on first use. Ensure:
- Internet connectivity
- Write permissions to StreamingAssets folder
- Firewall allows the download

### Video plays but no audio

**Causes:**
- Volume set to 0
- Audio track missing
- Unity audio settings

**Solutions:**
- Check `player.Volume`
- Verify stream has audio: `player.HasAudio`
- Check Unity's audio output device

### Stream timeout / won't open

**Causes:**
- Network issues
- Geo-restricted content
- Invalid URL

**Solutions:**
- Test URL in browser/VLC
- Check firewall/proxy
- Try different quality setting
- Increase timeout in player settings

### Poor performance / frame drops

**Solutions:**
- Use hardware acceleration (enabled by default)
- Reduce video quality
- Check CPU/GPU usage
- Disable other heavy operations during playback

## Platform Support

| Platform | Native Player | FFmpeg Player | URL Resolver |
|----------|---------------|---------------|--------------|
| Windows x64 | Media Foundation | Full | Full |
| macOS | AVFoundation | Full | Full |
| Linux x64 | GStreamer | Full | Full |
| iOS | AVFoundation | - | - |
| Android | MediaCodec | - | - |
| WebGL | - | - | - |

## Example Scenes

The package includes example scenes in `Samples~/`:

1. **BasicPlayer** - Simple video playback
2. **StreamPlayer** - HLS/DASH streaming
3. **TwitchPlayer** - Live Twitch stream with chat
4. **YouTubePlayer** - YouTube video with quality selection

Import via Package Manager > Prism Unity Plugin > Samples.

## License

See the LICENSE file in the package for details.

---

<div class="footer-nav">
  <a href="./">Home</a> |
  <a href="ffmpeg/">FFmpeg Plugin</a> |
  <a href="api/">API Reference</a>
</div>
