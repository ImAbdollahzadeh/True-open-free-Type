# True (open, free) Type
True (open, free) Type implementation from scratch

# Aims
1. Indroducing the concept of Bezier curve
2. Design of True (open, free) Type glyphs based on curves and lines
3. Rendering glyphs in subpixel regime
4. Implementation of smoothing low pass filters

# Prerequisites
- Knowledge of subpixel rendering on LCD panels
- Knowledge of C programming language

# Optimization
- SIMD optimization 

# Let's get started.
## Bezier Curves
What I did to render a curve on screen is constructing a power series (up to 3rd order) which control the curvature of the path.

In fact in my curve implementation, I stick to cubic bezier curve. It simply is an equation in form of ***A(X^3) + B(X^2) + CX + D***.

But it is not all. The design of a curve must follow having four points (a 2D coordination pair such as x and y), where point 0 and 3 are starting and ending points and points 1 and 2 are called control points. The latter points are the tuning of the curvature.

Having long concept short, I will create 4 2D points as an array (e.g. ***POINTS_2D curve_points[4]***) and change them in a way to render a desired curve for me.

For a detailed implementation and description of what I explained above, look at my source code (bezier_curve)

This is how it works in its native non-optimized way:

	static void draw_bezier_curve(POINT_2D* p, unsigned int color)
	{
		const unsigned int bits_per_pixel = 24;
		unsigned int bytes_per_pixel = bits_per_pixel >> 3;
		unsigned int width = WINDOW_WIDTH_ALIGN(bytes_per_pixel * 512); // statically, I want a window with size 512 x 512 pixels
		unsigned char red = color;
		unsigned char grn = (color & 0x00FF00) >> 8;
		unsigned char blu = (color & 0xFF0000) >> 16;
		double xu = 0.0; // x coordinate
  		double yu = 0.0; // y coordinate
  		double u  = 0.0; // iterator in range [0, 1]
  		
		for (u = 0.0; u <= 1.0; u += 0.001)
  		{
			// this is where all the curvature controlling coefficients are calculated
			double A = pow(1 - u, 3);
			double B = 3 * u * pow(1 - u, 2);
			double C = 3 * pow(u, 2) * (1 - u);
			double D = pow(u, 3);
			
			xu = A * p[0].x + B * p[1].x + C * p[2].x + D * p[3].x;
			yu = A * p[0].y + B * p[1].y + C * p[2].y + D * p[3].y;
			
			unsigned int px = ((unsigned int)yu * width) + ((unsigned int)xu * bytes_per_pixel);
			fb[px    ] = red;
			fb[px + 1] = grn;
			fb[px + 2] = blu;
		}
	}

As you might guess, there are quite some big steps to optmize the code above still in C. The coefficients A, B, C, and D can be calculated without further calling into function ***pow***. Another critical step would be acquiring x86-SSE assembly into play to further optimize xu and xy calculations and also casting them into unsigned int.
The detailed optimization would be neglected here for the sake of not getting too much complex. Of course later during my SIMD optimization tutorial, we will go through all the mentioned codes.

The following image shows a bezier curve rendered with the use of the upper source code. The four drawn small circles represent the place of 4 bezier curve control points.

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/bezier_demo.PNG"/>
</p>

## Designing a simple glyph based on only curves and lines
A glyph, this is what I would like to concentrate on how to build and render. For simplicity, I would stick to constant sized glyphs for all face-types. With that being said, I implement a 9 x 19 pixel area as a glyph to have a text in 12 pt size. To have something similar to 36 pt, for instance, I simply triple both width and height to have a 27 x 57 px area.

The 9 x 19 is not a random picked value, instead, I followed what had been already done in fonts such as consolas or Times New Roman. Therefore, my approximation is basically perfectly matched to the standard.

In my native C implementation, I consider each pixel to be 24 bits (3 bytes) long. This is, of course, not good for optimization, but truly fine for initial demonstration of the concept. Please note that, during the acceleration of the whole process of a glyph rendering, I will switch to 32 bits per pixel, but this would be left for future.

***What would be the actual pipeline towards the task? They are:***

1. drawing the borders of a face-type with the use of curves and lines
2. filling the pixels **inside** the borders to have a filled face-type (i.e. character)
3. distribute the designed face-type into subpixels (usage of single R-G-B elements of LCD panels)
4. calculating how much of each subpixel has been filled with the character to perform anti-aliasing
5. applying the RGB-YUV-RGB space conversions to filter (smooth) the character
6. rendering the result to the screen

If you look at the whole steps enumerated above, you'll notice that it takes the font rasterizer engine (our software) to be busy for *several ten miliseconds* to only render a single character on screen. As being said several times, for the purpose of demonstration, it is truly fine. Later on, I will get the hand dirty on how to accelerate everthing in the pipeline with a bit of C code modification and using x86-SSE optimization, therefore, stay tuned for the comming tutorials.

## STEP 1:
Design a working place. I scaled up my 9 x 19 pixelized area to 216 x 456 px area (24 times scaled up) to have a good look at what I am doing. Since I have 24 times scaling, it simply means, I assigned every 8 pixels to represent an R, G, or B subpixel element. Then I painted them accordingly and clarified the subpixels and pixels borders. You can have a look at the image in the following and try to count 9 x 19 px in my design.

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/RGB_template.PNG"/>
</p>

## STEP 2:
I drew some straight lines and curves to design the letter 'a'. It would be a lot of trial and errors to fix all the points to their positions and also to find a good-looking curve. I am quite happy with my letter 'a' and you can have a look at what I designed so far.

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/a_borders.PNG"/>
</p>

In my font_rasterizer source code, I have everything at hand too look at. As a side note, for later implementation of character filling, we need a way to signal our lines and curves. We need a flag for each line and curve to signal the engine what is the *direction* of the line or curve. You need to label them as being **UPWARD**, **DOWNWARD**, **RIGHTSIDE**, or **LEFTSIDE**. With this in mind, we would later be able to decide where to fill and where not to bother to fill in order to construct our face-type.

I assume it is a bit misleading and unclear now, what I explained regarding labeling each line or curve for its direction, but believe me, there is no other ways to decide where is **inside** a face-type and where is **outside**. Fine details come in the next step.

## STEP 3:
One of the most challenges in type-face design is to fill the pixels inside the borders defines with lines and curves.
For this reason we define a set of flags to label each element. 
There is one very important rule to be considered. Only pixels laying in between one element (i.e. line or curve) labeled as CONTOUR_DIRECTION_UP in its left side and an element with label CONTOUR_DIRECTION_DOWN in its right side are supposed to be painted. Think about an example for a moment (with the help of the following image). Suppose there is a standing rectangle that should be filled. We have two vertical lines and two horizontal lines to define the borders of this rectangle. The left vertical line was labeled as CONTOUR_DIRECTION_UP and the vertical line in the right side was CONTOUR_DIRECTION_DOWN. Whatever pixels are in between these 2 lines have to be filled. If there would be two rectangles next to each other, we are able to paint pixels between vertical lines of the leftmost rectangle, then there is pixels between right vertical line of the left rectangle and the left vertical line of the right rectangle that should be untouched. This can be only done since the labeling of lines now follows CONTOUR_DIRECTION_DOWN - CONTOUR_DIRECTION_UP and we invalidate this configuration for painting pixels. To make more sence of this, please look at the figure below and read this step once more from the beginnig.

	typedef enum _CONTOUR_DIRECTION {
		CONTOUR_DIRECTION_UP    = 0x1,
		CONTOUR_DIRECTION_DOWN  = 0x2,
		CONTOUR_DIRECTION_LEFT  = 0x3,
		CONTOUR_DIRECTION_RIGHT = 0x4,
	} CONTOUR_DIRECTION;

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/filling_basedOn_direction_flags.PNG"/>
</p>

Okay, now we know how to proceed. In general, when I design a face-type, I would think of givin a direction to it to be later filled up. The rule is to start from leftmost lines and curves towards right. This is bcause all the pixel's buffer reading and writing are done left to right. Having considered that, in my previous example with character 'a', I will have such a thing:

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/a_borders_with_directions.png"/>
</p>

Please note the points highlighted with black circles and also the directions that I assigned to each line or curve. If you search for a way to fill up only pixes **inside** the face-type, you would realize that the rule is to only paint pixels between UP-DOWN borders and all the rest cases should be neglected. Therefore, you see the easiness of using these CONTOUR_DIRECTION labels. 

One of the most critical steps in face-type rendering is ecatly this point, filling up the pixels inside a face-type border. You would scan line by line and keep track of pixels in between UP and DOWN labels and paint them with black. It is much more complex than what you think in terms of fast performance. In my 24 bits per pixel setup, there is less room for optimizations. One very obvious way is switching into 32 bits per pixel, hence, letting you paint pixels black with QWORD pointers (unsigned long long*) scanning instead of BYTE (unsigned char*) scanning. This will give you 8x speed up. Another trick which will be covered later during the SIMD optimization topic, would be aligning the whole face-type area to 16-bytes and fast scanning and filling pixels with XMM (i.e. 128 bit) pointers writing. This will also give you an extra 2x speed up. Therefore, you will be able to optimize pixel filling by roughly 16x faster speed than native C with BYTE pointer writing. Just to give you a sence of what we can achieve, I optimized the 'a' character filling up from ~5 ms to ~500 µs (10x faster).

This is time to actually apply all the criteria mentioned so far in this step, to construct our 'a' character. 
I have a shown a part of the source code from *font_rasterizer* here. It shows how one i able to implement the pixel filling up.

	static void fill_path(RASTERIZATION_MODE rm, BOOL subpixel)
	{
		if ((rm == RASTERIZATION_MODE_PATH_ONLY) && !subpixel)
			return;
		
		unsigned int i = 0, j = 0;
		while (j < wnd_ht)
		{
			while (i < wnd_wd)
			{
				if (fb_bitmap[(j * wnd_wd) + i] != 0)
				{
					unsigned char* framebuffer_pixel = &fb[(j*WINDOW_WIDTH_ALIGN(3 * wnd_wd)) + 3 * i];
					unsigned int   framebuffer_pixel_counter = 0;
					unsigned char* framebuffer_bitmap_pixel = &fb_bitmap[(j * wnd_wd) + i];
					
					/* add 1 to increment one byte */
					framebuffer_bitmap_pixel++;
					
					/* check to see that it is not the very last bitmap byte in a row */
					unsigned int cnt = wnd_wd - i;
					while ((*framebuffer_bitmap_pixel == 0) && cnt)
					{
						framebuffer_bitmap_pixel++;
						cnt--;
					}
					if (!cnt)
						break;
					
					/* if not the last bitmap byte in the row, do checking the acception direction conventions */
					framebuffer_bitmap_pixel = &fb_bitmap[(j * wnd_wd) + i];

					while ( !(*(framebuffer_bitmap_pixel + 1)) )
					{
						if (*framebuffer_bitmap_pixel == CONTOUR_DIRECTION_DOWN)
							break;

						unsigned char* target = framebuffer_bitmap_pixel + 1;
						while ( !(*target) )
							target++;

						framebuffer_pixel[framebuffer_pixel_counter    ] = 0;
						framebuffer_pixel[framebuffer_pixel_counter + 1] = 0;
						framebuffer_pixel[framebuffer_pixel_counter + 2] = 0;
						framebuffer_pixel_counter += 3;
						framebuffer_bitmap_pixel++;
					}

					/* very last pixel separately */
					framebuffer_pixel[framebuffer_pixel_counter    ] = 0;
					framebuffer_pixel[framebuffer_pixel_counter + 1] = 0;
					framebuffer_pixel[framebuffer_pixel_counter + 2] = 0;
				}
				i++;
			}
			i = 0;
			j++;
		}
	}

framebuffer_pixel is a byte representing the actual framebuffer, while framebuffer_bitmap_pixel is a byte representing the DIRECTION of each pixel in framebuffer_pixel.
During drawing a line or curve into framebuffer, I put the DIRECTION flag related to that line or curve into another separate buffer called framebuffer_bitmap_pixel. When my line or curve drawer decides to put a black pixel at, for intance, x = 100 and y = 50, at the same time it put a DIRECTION flag at x = 100 and y = 50 into framebuffer_bitmap_pixel. This is why in my source code provided above, ...
