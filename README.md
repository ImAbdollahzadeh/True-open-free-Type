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
In fact in my curve implementation, I stick to cubic bezier curve. It simply is an equation in form of AX^3 + BX^2 + CX + D.
But it is not all. The design of a curve must follow having four points (a 2D coordination pair such as x and y), where point 0 and 3 are starting and ending points and points 1 and 2 are called control points. The latter points are the tuning of the curvature.
Having long concept short, I will create 4 2D points as an array (e.g. POINTS_2D curve_points[4]) and change them in a way to render a desired curve for me.
