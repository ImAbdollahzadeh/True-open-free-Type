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

In fact in my curve implementation, I stick to cubic bezier curve. It simply is an equation in form of ***AX<sup>3</sup> + B<sup>2</sup> + CX + D***.

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

*framebuffer_pixel* is a byte representing the actual framebuffer, while *framebuffer_bitmap_pixel* is a byte representing the DIRECTION of each pixel in framebuffer_pixel.
During drawing of a line or curve into framebuffer, I put the **DIRECTION** flag related to that line or curve into another separate buffer called framebuffer_bitmap_pixel. When my line or curve drawer decides to put a black pixel at, for intance, x = 100 and y = 50, at the same time, it puts a DIRECTION flag at x = 100 and y = 50 into framebuffer_bitmap_pixel. 

	if (fb_bitmap[(j * wnd_wd) + i] != 0)
	
This part of the code searches for the first labeled pixel. When finds it, do the other criteria.

	/* check to see that it is not the very last bitmap byte in a row */
	unsigned int cnt = wnd_wd - i;
	while ((*framebuffer_bitmap_pixel == 0) && cnt)
	{
		framebuffer_bitmap_pixel++;
		cnt--;
	}
	if (!cnt)
		break;
		
If there is another pixel labeled in the same row, most probably we have an UP-DOWN combination, otherwise, if we would reach the end of that specific row, we simply neglect the rest of the row. In another word, to be a pixel inside a border, there should be two labeles in a given scan row. The left should be UP and the right must be DOWN (this is the only criteria to accept).
The variable ***cnt*** above makes us to see if we reach the end of the row or not. If ***cnt*** reaches 0, it means we covered the whole row without being able to find an UP-DOWN case to math. The rest of the source code is how to paint the pixels.

To demonstrate the power of the method, look at the following image.

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/a_letter_filled_completely.PNG"/>
</p>

## STEP 4:
### subpixel regime
So far we were able to have a filled character on our 9 x 19 px area (which is represetative of our glyph). Now the question is how to determine how much of a subpixel (R,G, or B elements of a pixel) has to be turned on in order to get the full advantage of subpixel rendeing regime?

For this reason, I have defined a data structure called **PIXEL_BLOCK** as following

	typedef struct _PIXEL_BLOCK {
		unsigned char    width;
		unsigned char    height;
		BLOCK_COLOR_CODE code;
		unsigned char    dark_pixels;
		unsigned char    color_intensity;
	} PIXEL_BLOCK;
	
In initialization of a PIXEL_BLOCK buffer, I would have

	void init_pixel_blocks(void)
	{
		unsigned int i;
		unsigned char code = 0;
		for (i = 0; i < (wnd_wd / 8) * (wnd_ht / 24); i++)
		{
			pixel_blocks[i].width           = 8;                      // actual row pixels of this block
			pixel_blocks[i].height          = 24;                     // actual column pixels of this block
			pixel_blocks[i].dark_pixels     = 192;                    // actual total number of pixels of this block
			pixel_blocks[i].color_intensity = 0xFF;                   // in the beginning our pixel block is fully turned on
			pixel_blocks[i].code            = (BLOCK_COLOR_CODE)code; // a flag to shows R, G, or B block
			code++;
			if (code == 3)
				code = 0;
		}
	}
	
A pixel block is one of the 8 x 24 colorful blocks (totally red, green, or blue) which I made to show a subpixel element (look at the given images above). If we look at the data structure, we have two important variables, *dark_pixels*, and *color_intensity*. During the initialization of pixel block buffer, dark_pixels field was given the value 192 (that is 8 times 24) and this is the total number of actual pixels in a pixel block. The variable color_intensity, on the other hand, gives the intensity of that particular pixel block and initially was set to 0xFF (fully turned on).

Okay, what we can get out of all this? If you look at the last image above, after having a filled 'a' character, many of the blocks are still fully uncovered with the character, some are partially covered, and some are fully black painted. This is where we need once more to go through some calculations to assign a final color intensity to every single pixel block based on the number of already covered pixels on that block.

	void put_pixel(unsigned int x, unsigned int y);
	
This function is called thousands of times by line or curve drawer functions to adjust the dark_pixels field of PIXEL_BLOCK buffer according to arguments x and y (the coordination of a given pixel). In this function, it finds the corresponding pixel block (with the use of x and y), and decrements the dark_pixels filed of that block.

	void put_block_on_fb(PIXEL_BLOCK* pb, unsigned int counter);
	
This function is the actual place where the engine re-draw all the pixel blocks from scratch based on their color_intensity field. The counter argument was given during ***rasterize_full_scene*** function call. It simply determines which pixel block we are in, and this parameter starts from 0 and goes to the last pixel block element.
Inside ***rasterize_full_scene*** function, we have

	while (counter < max)
	{
		pb = &pixel_blocks[counter];
		pb->color_intensity = (unsigned char)(1.328125f * (float)(pb->dark_pixels));
		put_block_on_fb(pb, counter);
		counter++;
	}
	
The second line inside the *while* loop, is where the pixel block color_intensity is calculated from the value dark_pixels for the block.
To see what we can get from all of these functions, have a look at the following image

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/a_letter_subpixel.PNG"/>
</p>

In the following, the real look of our subpixel font rendering at 12 pt and 36 pt is given for face types a, b, and i.

12 pt:

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/a_12.png"/>
</p>

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/b_12.png"/>
</p>

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/i_12.png"/>
</p>

36 pt:

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/a_36.PNG"/>
</p>

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/b_36.PNG"/>
</p>

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/i_36.PNG"/>
</p>

## STEP 5:
### RGB <-> YUV space transformation and facetype smoothing

If we carefully look at facetype 'a' rendered at 36 pt above, at the very convex side of the leter, at left most side, there is a red looking part which must be filtered (i.e. smoothing) in order to get better looked in our visual system. For this very important reason a color space transformation has o be applied.

One way to represent a pixel is through its own subpixels (R, G, and B elements). Another way is to look at this pixel in terms of red and blue chrominance as well as luminance factor. These elements are shown by u, v, and Y. Two elements u and v are a mixture of all colors (r, g, b) with different ballance in between. Y element, on the other hand, is the intensity, or brightness of the whole pixel. There is a simple transformation as:

	y =  0.2990 * r + 0.5870 * g + 0.1140 * b;
	u = -0.1471 * r - 0.2888 * g + 0.4360 * b;
	v =  0.6150 * r - 0.5149 * g - 0.10001 * b;

If we look at the transformation relation, we will see that the Y element mostly comes from R and G elements (green-yellowish brightness), u comes from B element which somwhow considers the blue color of the pixel, and v mainly considers redness. 

Te next step is to keep Y constant (because we don't want to lose the intensity or energy of the pixel), anddecrease the u and v elements in order to lower the chrominance of the corresponding pixel. Here we apply a 50 % decrease (you can experiment another value).

	u *= 0.5;
	v *= 0.5;
	
and do the inverse transformation to RGB color space with new u and v values like:

	r = y + (1.13983 * v);
	g = y - (0.39465 * u) - (0.5806 * v);
	b = y + (2.03211 * u);

The complete non-optimized source code for such a color space transformation is in function **save_subpixel_scene**

	/* check yuv. If so, perform it */
	if (yuv)
	{
		// ...
		unsigned int  alignment = 0;
		unsigned int  px        = 0;
		unsigned char red       = 0;
		unsigned char grn       = 0;
		unsigned char blu       = 0;
		double y = 0;
		double u = 0;
		double v = 0;
		
		unsigned int row_counter = 0;
		unsigned int alignment_step = WINDOW_WIDTH_ALIGN(3 * (wnd_wd) / 24) - (3 * (wnd_wd) / 24);

		while (px < (3 * (wnd_wd) / 24) * (wnd_ht / 24))
		{
			red = output_buffer[px + alignment + 0];
			grn = output_buffer[px + alignment + 1];
			blu = output_buffer[px + alignment + 2];
		
			double r = (double)red;
			double g = (double)grn;
			double b = (double)blu;
		
			y = 0.299*r + 0.587*g + 0.114*b;
			u = -0.1471*r - 0.2888*g + 0.436*b;
			v = 0.615*r - 0.5149*g - 0.10001*b;
			u *= 0.5;
			v *= 0.5;
			r = y + (1.13983 * v);
			g = y - (0.39465 * u) - (0.5806 * v);
			b = y + (2.03211 * u);
		
			yuv_buffer[px + alignment + 0] = (unsigned char)r;
			yuv_buffer[px + alignment + 1] = (unsigned char)g;
			yuv_buffer[px + alignment + 2] = (unsigned char)b;
		
			px += 3;
			row_counter++;
			if (row_counter == ((wnd_wd) / 24))
			{
				row_counter = 0;
				alignment += alignment_step;
			}
		}
		// ...
	}
In the following image, the effect of such transformation was shown. Note that the filtering was 50 % uv decreasing ad keeping y constant.

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/yuv_filter.PNG"/>
</p>

and in its true size (36 pt) it looks something like:

<p align="center">
	<img src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/yuv_filter_2.PNG"/>
</p>

Th bright red pixels (at leftmost) and glowing blue pixels at rightmost sides, now, after yuv filtering, are gone away.
