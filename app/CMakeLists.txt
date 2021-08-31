add_subdirectory(videoio)
add_executable(backscrub deepseg.cc)
target_link_libraries(backscrub
    libbackscrub
    videoio
    opencv_core
    opencv_video
    opencv_videoio
    opencv_imgproc
    opencv_imgcodecs
    opencv_highgui
)
install(TARGETS backscrub)