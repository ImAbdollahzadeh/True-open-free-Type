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
