# AzuraCast Liquidsoap YouTube Video Stream Script

This script is designed for use with AzuraCast and Liquidsoap, enabling a video stream to YouTube using a static MP4 video file. The script is intended for installations using Docker.

## Features

- Loops a specified MP4 video while overlaying "Now Playing" text.
- Streams directly to YouTube with customizable settings.
- Easy to configure for your station.

## Prerequisites

- A running Docker installation of AzuraCast.
- A valid YouTube stream key.
- A static MP4 video file for streaming.
- A TTF font file for the "Now Playing" text.


3. **Copy and Paste the Script**  
Open the `liquidsoap.conf` file and paste the following script at the desired location:

```lua
# VIDEO STREAM 

# Edit This: Station Base Directory
station_base_dir = "/var/azuracast/stations/your_station"

# Edit This: YouTube Stream Key
youtube_key = "your-youtube-stream-key"

# Path to the video file that will loop behind the Now Playing text (you have to provide this)
video_file = station_base_dir ^ "/media/video/your-video-file.mp4"

# Path to a font (TTF) file that will be used to draw the Now Playing text (you have to provide this)
font_file = station_base_dir ^ "/media/video/your-font-file.ttf"

# A static file auto-generated by AzuraCast in the "config" dir.
nowplaying_file = station_base_dir ^ "/media/video/nowplaying.txt"

# Align text
font_size = "50"
font_x = "340"
font_y = "990"
font_color = "white"

# Method to overlay now playing text
def add_nowplaying_text(s) =
  def mkfilter(graph)
    let {video = video_track} = source.tracks(s)
    video_track = ffmpeg.filter.video.input(graph, video_track)
    video_track = ffmpeg.filter.drawtext(fontfile=font_file, fontsize=font_size, x=font_x, y=font_y, fontcolor=font_color, textfile=nowplaying_file, reload=5, graph, video_track)
    video_track = ffmpeg.filter.video.output(graph, video_track)

    source({
      video = video_track
    })
  end

  ffmpeg.filter.create(mkfilter)
end

videostream = single(video_file)
videostream = add_nowplaying_text(videostream)
videostream = source.mux.video(video=videostream, radio)

# Output to YouTube
enc = %ffmpeg(
    format="mpegts", 
    %video.raw(codec="libx264", pixel_format="yuv420p", b="300k", preset="superfast", r=25, g=50),
    %audio(
        codec="aac",
        samplerate=44100,
        channels=2,
        b="120k",
        profile="aac_low"
    )
)

output.youtube.live.hls(key=youtube_key, fallible=true, encoder=enc, videostream)
