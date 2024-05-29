# Displaying Images on the SSD-1306 OLED Display with MicroPython
The **SSD-1306** is a popular driver chip used in small monochrome OLED displays. It is widely used in various electronic projects and consumer electronics due to its simplicity and effectiveness in driving OLED panels. The SSD-1306 commonly comes in resolutions of 128x64 pixels (128 wide, 64 high).

**MicroPython** is an implementation of the Python 3 programming language designed to run on microcontrollers and other resource-constrained environments. It provides a subset of the Python standard library, tailored to the requirements of microcontroller hardware.

I developed a developed a system for converting any bitmap image (JPG, PNG, etc.) to a format that can be displayed on the SSD-1306. This repo contains both the code to this system so you can use it yourself, and an explanation as to how it works with the SSD-1306.

## Getting Up and Running: Basic SSD-1306 Interfacing
The [`ssd1306.py` module](./src/ssd1306.py) provides an excellent class for interfacing with the SSD1306 via I2C. I did not write this code; in fact, I do not know who did as the file itself does not credit a developer. I found this file in [this video](https://www.youtube.com/watch?v=oaM80GyVIwA) but also in other areas online. I am unsure of the author.

With that loaded onto your MicroPython board (I am using a Raspberry Pi Pico, RP2040), you can display on the SSD-1306 OLED display like this:

```
import machine
import ssd1306
import time

# create I2C interface
i2c = machine.I2C(1, sda=machine.Pin(14), scl=machine.Pin(15)) # I have my display hooked up to pins 14 and 15 (for I2C)
print(i2c.scan()) # 0x3c is the I2C address of the SSD1306. As an integer, 60.

# create SSD1306 interface
display_width:int = 128
display_height:int = 64
oled = ssd1306.SSD1306_I2C(128, 64, i2c)

# turn on all pixels
oled.fill(1) # make the change
oled.show() # show the update
time.sleep(1)

# turn off all pixels (blank display)
oled.fill(0)
oled.show()
time.sleep(1)

# fill in a single pixel
oled.pixel(0, 0, 1) # turn on pixel at (0, 0) (top left)
oled.pixel(64, 0, 1) # turn on pixel at (64, 0), middle top
oled.pixel(127, 63, 1) # turn on pixel at (127, 63) (absolute bottom right on my 128x64 display)
oled.show() # show the update
time.sleep(1)

# turn off all pixels again
oled.fill(0)
oled.show()
time.sleep(1)

# display text
oled.text("Hello, world!", 0, 0) # print text "Hello, world!" at position (0, 0) (top left)
oled.show()
time.sleep(1)

# turn off all pixels again
oled.fill(0)
oled.show()
time.sleep(1)
```

The above example demonstrates basic manipulation of the OLED display. The example below demonstrates (just add this to the bottom of the file below) how to display an image (represented as a bytearray, but that is explained later):

```
import framebuf

# Load smiley face image and display
# The below bytearray is a buffer representation of a 32x32 smiley face image. This is explained in this documentation below - continue reading!
smiley = bytearray(b'\x00?\xfc\x00\x00\xff\xff\x00\x03\xff\xff\xc0\x07\xe0\x07\xe0\x0f\x80\x01\xf0\x1f\x00\x00\xf8>\x00\x00|<\x00\x00<x\x00\x00\x1epx\x1e\x0e\xf0x\x1e\x0f\xe0x\x1e\x07\xe0x\x1e\x07\xe0\x00\x00\x07\xe0\x00\x00\x07\xe0\x00\x00\x07\xe1\xc0\x03\x87\xe1\xc0\x03\x87\xe1\xc0\x03\x87\xe1\xe0\x07\x87\xe0\xf0\x0f\x07\xf0\xf8\x1f\x0fp\x7f\xfe\x0ex?\xfc\x1e<\x0f\xf0<>\x00\x00|\x1f\x00\x00\xf8\x0f\x80\x01\xf0\x07\xe0\x07\xe0\x03\xff\xff\xc0\x00\xff\xff\x00\x00?\xfc\x00')
fb = framebuf.FrameBuffer(smiley, 32, 32, framebuf.MONO_HLSB) # load the 32x32 image binary data in to a FrameBuffer
oled.blit(fb, 0, 0) # project or "copy" the loaded smiley image FrameBuffer into the OLED display at position 0,0 (the top left of the smiley will touch the top left corner of the display)
oled.show()
```

## How does displaying an image on the SSD-1306 work?
When it comes down to it, the basic OLED display represents each pixel of the display as either a **0** or a **1**. A **0** means that the pixel is off while a **1** means that a pixel is on. Consider the following 8x8 image of an "X", for example: 

![8x8](https://i.imgur.com/xxWlOUD.png)

You can see that in this image of 64 pixels (8 pixels wide, 8 pixels high), the dark pixels (filled in) are represented by a 1 while the light pixels (not filled in) are represented by a 0.

As you probably know, a **byte** is a sequence of **eight bits** (8 zeros and ones). Observing the image above, we can see that, from left to right, top to bottom, we can "chunk" this image into **eight** bytes, each consisting of **eight bits**. For example, the first byte fits cleanly across the top row!

![first row](https://i.imgur.com/swr5YJR.png)

Each subsequent row of 8 bits can also be converted to a single byte form:

![all rows](https://i.imgur.com/3p5lhCr.png)

Thus, the entire 8x8 image above can be represented as a sequence of 8 bytes (from top to bottom):

```
129,66,36,24,24,36,66,129
```

In python, added to a `bytearray`, that will print into a `str` that looks like this:
```
b'\x81B$\x18\x18$B\x81'
```

We an extrapolate this understanding to broader examples. In the above simple example, the image neatly translated into 8 rows of 8 bits, each row representing a byte. For larger images, the same process occurs, sometimes there being multiple bytes per row or sometimes there even being a portion of a byte per row (from left to right, top to bottom).

### So how can I convert a JPG/PNG image I have to buffer representation?
I wrote code for doing that! You can find that code as the `image_to_buffer` *def* in the [`convert.py` file](./src/convert.py). This code:
1. Uses [**pillow**](https://pypi.org/project/pillow/) to open an image.
2. Loops through each RGB value of the image and decides if the pixel should be considered "on" or "off" based on a threshold.
3. Assembles a list of bits representing each pixel.
4. Chunks these bits into groups of 8.
5. Converts each group of 8 bits into a byte.
6. Collects these converted bytes and returns them to you - ready to be shown on the SSD-1306 display!

For example, I made a simple 128x64 (128 pixels wide, 64 pixels high) JPG image in Paint [here](./hello_world.jpg):

![hello world](./hello_world.jpg)

To convert this to a buffer representation that can be loaded into the SSD-1306:

```
>>> import convert
>>> converted = convert.image_to_buffer(r"C:\Users\timh\Downloads\oled\hello_world.jpg")
>>> buffer = converted[0]
>>> buffer
b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00|\x03\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xfe\x0f\x80\x00\x00\x00\x00\x00\x00\x00@\x00\x00\x00\x00\x01\xff?\xc0\x00\x00\x00\x00\x00\x00\x00\xe0\x00\x00\x00\x00\x01\xe7\xff\xc0\x00\x00\x00\x04\x00\x00\x00\xe0\x00\x00\x00\x00\x01\xc7\xf9\xc0\x00\x00\x10\x0e\x00\x00\x00\xe0\x00\x00\x00\x00\x01\xc7\xe1\xc0\x00\x008\x0e\x00\x00\x00\xe1\x00\x00\x00\x00\x01\xc3\xc1\xc0\x00\x008\x0e\x00\x00\x00\xe3\x80\x00\x00\x00\x01\xc0\x01\xc0\x00\x008\x0e\x00\x00\x00\xe3\x80\x00\x00\x00\x01\xc0\x03\xc0\x00\x00x\x0e\x00\x00\x00\xe3\x80\x00\x00\x00\x01\xc0\x03\x80\x00\x00p\x0e\x00\x00\x00\xe3\x80\x00\x00\x00\x01\xc0\x07\x80\x00\x00p\x0e\x01\xfc\x00\xe3\x80\x00\x00\x00\x01\xe0\x0f\x80\x00\x00p\x0f\xc3\xfe\x00\xe3\x80\x00\x00\x00\x00\xf0?\x00\x00\x00\xf7\xff\xe7\xfe\x00\xe3\x80\x07\x00\x00\x00\x7f\xfe\x00\x00\x00\xef\xff\xef\x8e\x00\xe3\x80\x0f\x80\x00\x00?\xfc\x00\x00\x00\xe7\xfe\x0f\x0e\x00\xe3\x80\x1f\x80\x00\x00\x1f\xf0\x00\x00\x00\xe0\x1c\x1e\x1e\x00\xe3\x80?\x80\x00\x00\x0f\x00\x00\x00\x00\xe0\x1c\x1f\xfe\x00\xe3\x80\x7f\x80\x00\x00\x00\x00\x00\x00\x00\xe0<\x1f\xfc\x00\xe3\x80\xf3\x80\x00\x00\x00\x00\x00\x00\x01\xe08\x1f\xf8\x00\xe3\x81\xe3\x80\x00\x00\x00\x00\x00\x00\x01\xc08\x0f\x00\x00\xe3\x81\xe3\x80\x00\x00\x00\x00\x00\x00\x01\xc08\x07\xe0\x00\xe3\x81\xc3\x80\x00\x00\x00\x00\x00\x00\x01\xc08\x03\xfc@\xe3\x81\xc3\x80\x00\x00\x00\x00\x00\x00\x01\xc08\x01\xff\xe0c\x81\xc7\x80\x00 \x00\x00\x00\x00\x01\xc0x\x00?\xe0\x01\x81\xcf\x00\x00p\x00\x00\x00\x00\x01\xc0x\x00\x07\xe0\x00\x01\xff\x00\x00p\x00\x00\x00\x00\x00\xc00\x00\x00\x00\x00\x01\xff\x00\x00\xf0\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xf8\x00\x00\xf0\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00@\x00\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xe0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xe0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xe0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01\xe0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01\xc0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x04\x01\xc0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x04\x00\x00\x00\x0e\x01\xc0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x0e\x00\x00\x00\x1e\x01\xc0\x01\xf0\x03\x80\x00\x00\x00\x00\x0c\x0e\x00\x0c\x00\x1e\x01\xc0\x03\xf0\x03\x80\x00\x00\x18\x00\x1e\x1e\x00\x1f\x00\xfc\x01\xc0\x03\xf0\x03\x80\x00\x00<\x00>\x1c\x00\x7f\x81\xf8\x01\xc0\x03\xf0\x03\x80\x00\x00\x1c\x00~<\x00\xff\x83\xf8\x01\xc0\x03\xf0\x03\x80\x00\x00\x1e\x00~8\x01\xfb\x83\xe0\x01\xc0\x07\xe0\x03\x80\x00\x00\x0e\x00~8\x03\xe3\x83\xc0\x01\xc0\x1f\xe0\x03\x80\x00\x00\x0f\x00\xfex\x07\xc3\x83\xc0\x01\xc0?\xe0\x03\x80\x00\x00\x07\x00\xefp\x0f\x03\x83\x80\x01\xc0\x7f\xe0\x03\x80\x00\x00\x07\x81\xe7\xf0\x0f\x03\x87\x80\x01\xc0\xff\xe0\x03\x80\x00\x00\x03\x83\xc7\xe0\x0e\x07\x87\x00\x01\xc1\xef\xe0\x03\x80\x00\x00\x03\xc3\xc7\xe0\x1e\x07\x07\x00\x01\xc3\xee\xe0\x01\x80\x00\x00\x01\xe7\x87\xc0\x1c\x0f\x07\x00\x01\xc3\xce\xe0\x00\x00\x00\x00\x01\xef\x87\xc0\x1c\x0f\x07\x00\x01\xc3\x8e\xe0\x00\x00\x00\x00\x00\xfe\x07\xc0\x1c\x1e\x07\x00\x01\xc7\x8e\xe0\x00\x00\x00\x00\x00~\x03\x80\x1e>\x07\x00\x01\xc7\x1e\xe0\x00\x00\x00\x00\x00>\x00\x00\x0f\xfc\x07\x00\x01\xc7<\xe0\x00\x00\x00\x00\x00\x1c\x00\x00\x0f\xf8\x03\x00\x01\xc7|`\x01\x00\x00\x00\x00\x00\x00\x00\x07\xf0\x00\x00\x01\xc7\xf8\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xc7\xf0\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x03\xc0\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00'
```

The `image_to_buffer` def above opens the bitmap image and translates it to a buffer representation that can be displayed on the SSD-1306. The `image_to_buffer` function returns three values as a tuple - firstly, the buffer, then the width and height of the image. Technically, these aren't necessarily needed if you already know the dimensions of your image, but they are returned here as well for convience purposes as these two values will be required to load the buffered image into memory in MicroPython.

### Load to the device!
Like in the example earlier in this document, that buffer can now be loaded into memory in the device controlling the SSD-1306 via MicroPython:

```
# load buffer
hello_world_buf = bytearray(b'\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00|\x03\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xfe\x0f\x80\x00\x00\x00\x00\x00\x00\x00@\x00\x00\x00\x00\x01\xff?\xc0\x00\x00\x00\x00\x00\x00\x00\xe0\x00\x00\x00\x00\x01\xe7\xff\xc0\x00\x00\x00\x04\x00\x00\x00\xe0\x00\x00\x00\x00\x01\xc7\xf9\xc0\x00\x00\x10\x0e\x00\x00\x00\xe0\x00\x00\x00\x00\x01\xc7\xe1\xc0\x00\x008\x0e\x00\x00\x00\xe1\x00\x00\x00\x00\x01\xc3\xc1\xc0\x00\x008\x0e\x00\x00\x00\xe3\x80\x00\x00\x00\x01\xc0\x01\xc0\x00\x008\x0e\x00\x00\x00\xe3\x80\x00\x00\x00\x01\xc0\x03\xc0\x00\x00x\x0e\x00\x00\x00\xe3\x80\x00\x00\x00\x01\xc0\x03\x80\x00\x00p\x0e\x00\x00\x00\xe3\x80\x00\x00\x00\x01\xc0\x07\x80\x00\x00p\x0e\x01\xfc\x00\xe3\x80\x00\x00\x00\x01\xe0\x0f\x80\x00\x00p\x0f\xc3\xfe\x00\xe3\x80\x00\x00\x00\x00\xf0?\x00\x00\x00\xf7\xff\xe7\xfe\x00\xe3\x80\x07\x00\x00\x00\x7f\xfe\x00\x00\x00\xef\xff\xef\x8e\x00\xe3\x80\x0f\x80\x00\x00?\xfc\x00\x00\x00\xe7\xfe\x0f\x0e\x00\xe3\x80\x1f\x80\x00\x00\x1f\xf0\x00\x00\x00\xe0\x1c\x1e\x1e\x00\xe3\x80?\x80\x00\x00\x0f\x00\x00\x00\x00\xe0\x1c\x1f\xfe\x00\xe3\x80\x7f\x80\x00\x00\x00\x00\x00\x00\x00\xe0<\x1f\xfc\x00\xe3\x80\xf3\x80\x00\x00\x00\x00\x00\x00\x01\xe08\x1f\xf8\x00\xe3\x81\xe3\x80\x00\x00\x00\x00\x00\x00\x01\xc08\x0f\x00\x00\xe3\x81\xe3\x80\x00\x00\x00\x00\x00\x00\x01\xc08\x07\xe0\x00\xe3\x81\xc3\x80\x00\x00\x00\x00\x00\x00\x01\xc08\x03\xfc@\xe3\x81\xc3\x80\x00\x00\x00\x00\x00\x00\x01\xc08\x01\xff\xe0c\x81\xc7\x80\x00 \x00\x00\x00\x00\x01\xc0x\x00?\xe0\x01\x81\xcf\x00\x00p\x00\x00\x00\x00\x01\xc0x\x00\x07\xe0\x00\x01\xff\x00\x00p\x00\x00\x00\x00\x00\xc00\x00\x00\x00\x00\x01\xff\x00\x00\xf0\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xf8\x00\x00\xf0\x01\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00@\x00\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xe0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xe0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xe0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01\xe0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x01\xc0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x00\x00\x00\x00\x04\x01\xc0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x04\x00\x00\x00\x0e\x01\xc0\x01\xf0\x03\x80\x00\x00\x00\x00\x00\x0e\x00\x00\x00\x1e\x01\xc0\x01\xf0\x03\x80\x00\x00\x00\x00\x0c\x0e\x00\x0c\x00\x1e\x01\xc0\x03\xf0\x03\x80\x00\x00\x18\x00\x1e\x1e\x00\x1f\x00\xfc\x01\xc0\x03\xf0\x03\x80\x00\x00<\x00>\x1c\x00\x7f\x81\xf8\x01\xc0\x03\xf0\x03\x80\x00\x00\x1c\x00~<\x00\xff\x83\xf8\x01\xc0\x03\xf0\x03\x80\x00\x00\x1e\x00~8\x01\xfb\x83\xe0\x01\xc0\x07\xe0\x03\x80\x00\x00\x0e\x00~8\x03\xe3\x83\xc0\x01\xc0\x1f\xe0\x03\x80\x00\x00\x0f\x00\xfex\x07\xc3\x83\xc0\x01\xc0?\xe0\x03\x80\x00\x00\x07\x00\xefp\x0f\x03\x83\x80\x01\xc0\x7f\xe0\x03\x80\x00\x00\x07\x81\xe7\xf0\x0f\x03\x87\x80\x01\xc0\xff\xe0\x03\x80\x00\x00\x03\x83\xc7\xe0\x0e\x07\x87\x00\x01\xc1\xef\xe0\x03\x80\x00\x00\x03\xc3\xc7\xe0\x1e\x07\x07\x00\x01\xc3\xee\xe0\x01\x80\x00\x00\x01\xe7\x87\xc0\x1c\x0f\x07\x00\x01\xc3\xce\xe0\x00\x00\x00\x00\x01\xef\x87\xc0\x1c\x0f\x07\x00\x01\xc3\x8e\xe0\x00\x00\x00\x00\x00\xfe\x07\xc0\x1c\x1e\x07\x00\x01\xc7\x8e\xe0\x00\x00\x00\x00\x00~\x03\x80\x1e>\x07\x00\x01\xc7\x1e\xe0\x00\x00\x00\x00\x00>\x00\x00\x0f\xfc\x07\x00\x01\xc7<\xe0\x00\x00\x00\x00\x00\x1c\x00\x00\x0f\xf8\x03\x00\x01\xc7|`\x01\x00\x00\x00\x00\x00\x00\x00\x07\xf0\x00\x00\x01\xc7\xf8\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\xc7\xf0\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x03\xc0\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00\x00')

import framebuf
hello_world_img = framebuf.FrameBuffer(hello_world_buf, 128, 64, framebuf.MONO_HLSB) # load as an image 128 pixels wide, 64 pixels high
oled.blit(hello_world_img, 0, 0) # write the 128x64 image to the SSD-1306 display, positioned in the top left corner
oled.show() # show on the display
```

And the [hello world image](./hello_world.jpg) is displayed!

![hello world](https://i.imgur.com/51eorVj.png)

The buffer is "hard coded" within the code above. However, the binary data itself can just as easily be stored in file form on-device!

## Graphic Collections
https://www.flaticon.com/packs/alphabet-and-numbers-11