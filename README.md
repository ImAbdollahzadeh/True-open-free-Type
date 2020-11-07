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

As you might guess, there are quite some big steps to optmize the code above still in C. The coefficients A, B, C, and D can be calculated without further calling ino function ***pow***. Another critical step would be acquiring x86-SSE assembly into play to further optimize xu and xy calculations and also casting them into unsigned int.
The detailed optimization would be neglected hre for the sake of not getting too much complex. Of course later during my SIMD optimization tutorial, we will go through all the mentioned codes.

The following image shows an example bezier curve that I rendered with the use of above mentioned source code. The 4 dawn small circles represent te place of 4 bezier curve control points.

<img align="center" width="300" height="300" src="https://github.com/ImAbdollahzadeh/True-open-free-Type/blob/main/tutorial_resources/bezier_demo.PNG"/>
