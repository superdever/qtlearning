# use copilot generate a widget application by follow text

# create a VideoPlayer widget application, the UI should save to .ui file
# main(...) function will start the MainForm window, the default size is 800*600.
# after MainForm opened, user can click "Browse..." button, and then user can select multiple video files,and then click "OK" to select the video files.
# MainForm will play all of the selected videos, each video will be played in a fixed area in MainForm, the video area size is about 150*150. the MainForm layout is flow layout, the video area should display from left to right, top to bottom.
# the video area should be a customer widget, named ItemVideoWidget, it's used to play the single specified video, it should use ffmpeg to decode and use opengl to accellence performance.
# in MainForm, when right click the video area, the context menu will display, it contain Start and Stop functions, when click Start to play video, when click Stop to stop play video. Start button should be disabled if video is played. Stop button should be disabled if video is stopped.
# in MainForm, when double click the video area, it will pop up a new VideoWidget, it will continue to play the video which be double-clicked. the VideoWidget size should be auto ajust to same with the video orignal size, not the fixed size 150*150.
# in VideoWidget, the toolbar should display, it contains Start,Stop,Screenshot, ShowText etc. start and stop to control the video wheather playing, ShowText will input some text to play on the left corner of the video, color is red.
# in MainForm, user can double click video 1 to open the VideoWidget, also can click video 2 to open the VideoWidget, but at a same time, only 1 VideoWidget instance exists.
