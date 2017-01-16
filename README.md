# gjk.c – Gilbert-Johnson-Keerthi in plain C
This is a rough but fast implementation of GJK collision detection algorithm in plain C. Just one C file, less then 200 lines, no dependencies. It is in 2D for now, full 3D version is upcoming... This 2D-version uses Minkowski sums and builds a triangle-simplex in Minkowski space to tell if two arbitrary convex polygons are colliding. 3D-version will be roughly the same, but will build a tetrahedron-simplex inside a 3-dimensional Minkowski space. It currently only tells if there is a collision or not. C-code for distance and contact points coming soon.

## Disclaimer
Fuck licenses and copyright. I made it for learning purposes, this is public knowledge and it's absolutely free for any usage.

## Usage example
This is an illustration of the example case from [dyn4j](http://www.dyn4j.org/2010/04/gjk-gilbert-johnson-keerthi/).

![Example case from dyn4j](http://www.dyn4j.org/wp-content/uploads/2010/04/gjk-figure1.png "Example case from dyn4j")

The two tested polygons are defined as arrays of plain C vector struct type. This implementation of GJK doesn't really care about the order of the vertices in the arrays, as it treats all sets of points as *convex polygons*.

```c
struct _vec2 { float x; float y; };
typedef struct _vec2 vec2;

...

int main(int argc, const char * argv[]) {
    
    // test case from dyn4j

    vec2 vertices1[] = {
        { 4, 11 },
        { 4, 5 },
        { 9, 9 },
    };
    
    vec2 vertices2[] = {
        { 5, 7 },
        { 7, 3 },
        { 10, 2 },
        { 12, 7 },
    };

    size_t count1 = sizeof (vertices1) / sizeof (vec2); // == 3
    size_t count2 = sizeof (vertices2) / sizeof (vec2); // == 4
    
    int collisionDetected = gjk (vertices1, count1, vertices2, count2);
    
    printf (collisionDetected ? "Bodies collide!\n" : "No collision\n");
    return 0;
}
```
## GJK Explanation

At the very top level the goal of GJK algorithm is to tell if two arbitrary shapes are intersecting (colliding) or not right now. A collision occurs when two shapes try to occupy the same points in space. The space can be of any nature. It might be your in-game world simulation, or a calculation on a table of statistical data or any other application you can imagine. You can have as many dimensions as you want, the amount of dimensions does not really matter, the logic is the same for 1D, 2D, 3D, etc... With GJK you can even calculate collisions in 4D if you're able to comprehend this. The algorithm itself does not require any hard math whatsoever, it's actually very intuitive and easy. It takes an arithmetic difference of two shapes by subtracting all points of one shape from all points of another shape. If two shapes have a common point in them, subtracting that point from itself results in a difference of zero. So, if zero is found in the resulting difference of two shapes then there's a collision. In fact, the algorithm does not have to calculate all differences for every single pair of points, only a small subset of significant points is examined, therefore it works very fast while being very accurate.

In order to understand GJK one has to build an imaginary visualization of what is going on under the hood. Once you see the picture in your head, you can implement it and even tweak it for your needs. 

Let's start with a naive example of computing a shape difference in one dimension. A segment of a number line is the simplest 1D-shape. Imagine we have two segments on the number line: segment `[1,3]` and segment `[2,4]`:
```
·····O·····1=====2=====3·····+·····+·····+·····> x

·····O·····+·····2=====3=====4·····+·····+·····> x
```
Zero is our point of reference on the number line, so we call it *the Origin*. Easy enough.
It is obvious that our segments occupy some common region of our 1D-space, so they must be intersecting or colliding, you can tell that just by looking at the representation of both segments in same 1D-space (on the same line). Let's confirm arithmetically that these segments indeed intersect. We do that by subtracting all points of segment `[2,3]` from all points of segment `[1,3]` to see what we get on a number line.
```
1 - 2 = -1
1 - 3 = -2
1 - 4 = -3
2 - 2 =  0
2 - 3 = -1
2 - 4 = -2
3 - 2 =  1
3 - 3 =  0
3 - 4 = -1
```
We got a lot of numbers, many of them more than once. The resulting set of points (numbers) is larger than each of the original sets of points of two shapes. Let's plot these resulting points in our 1D-space and look at the shape of resulting segment on the number line:
```          
·····+····-3====-2====-1=====O=====1·····+·····+·····> x
```
So after all we got resulting segment `[-3,1]` which covers points `-3`, `-2`, `-1`, `0` and `1`. Because two initial shapes had some points in common the resulting segment contains a zero. This comes from a simple fact, that when you subtract a point (a number) from itself you inevitably end up with a zero. Note, that our initial segments had points 2 and 3 in common. When we subtracted 2 from 2 we got 0. When we subtracted 3 from 3 we also got a zero. This is quite obvious. So, if two shapes have at least one common point, because you subtract it from itself, the resulting set must contain the zero point (the Origin) at least once. This is the key of GJK which says: if the Origin is contained inside the resulting set – the original shapes must have collided or kissed at least. Once you get it, you can then apply it to any number of dimensions.

Now let's take a look at a counter-example, say we have two segments `[-2,-1]` and `[1,3]`:
```
·····+····-2====-1·····O·····+·····+·····+·····+·····> x

·····+·····+·····+·····O·····1=====2=====3·····+·····> x
```
We can visually ensure that segment `[-2,-1]` occupies a different region of number line than that of segment `[1,3]`, so these two shapes do not intersect in our 1D-space, therefore there's no collision. Let's prove that arithmetically by subtracting all points of any of the two segments from all points of the other.
```
1 - (-1) = 1 + 1 = 2
1 - (-2) = 1 + 2 = 3
2 - (-1) = 2 + 1 = 3
2 - (-2) = 2 + 2 = 4
3 - (-1) = 3 + 1 = 4
3 - (-2) = 3 + 2 = 5 
```
And we again draw the resulting segment on a number line which is our imaginary 1D-space:
```
·····+·····O·····+·····2=====3=====4=====5·····+·····> x
```
We got another bigger segment `[2,5]` which represents a difference of all the points of the original two segments but this time it does not contain the Origin. That is, the resulting set of points does not include zero, because original segments did not have any points in common so they indeed occupy different regions of our number line and don't intersect.

Now, if our initial shapes were too big (long initial segments) we would have to calculate too many differences from too many pairs of points. But it's actually easy to see, that we only need to calculate the difference between the endpoints of two segments, ignoring all the 'inside' points of both segments.

Consider two segments `[10,20]` and `[5,40]`:
```
·····O····10====20·····+·····+·····+·····+·····> x

·····O··5==+=====+=====+====40·····+·····+·····> x
```
Now subtract four endpoints from each other:
```
10 - 5  =   5
20 - 5  =  15
10 - 40 = -30
20 - 40 = -20
```
The resulting segment `[-30,15]` would look like this:
```
·····+···-30=====+=====+=====O=====+=15··+·····> x
```
We ignored all insignificant internal points and only took the endpoints of original segments into account thus reducing our calculation to four basic arithmetic operations (subtractions). We did that by switching to a simpler representation of a segment (only two endpoints instead of all points contained inside an original segment). This simpler representation of the difference of two shapes is called a 'simplex'. It literally means 'the simplest possible'. A segment is indeed the simplest possible shape which is sufficient to contain points of a number line in order to determine if two original segments occupy some common region of 1D-space. Even if one segment covers the other in its entirety (one segments fully contains the other segment) – you can still detect an intersection of them in space. And it does not matter which one you're subtracting from – the resulting set will still contain the Origin at zero.

GJK also works if two segments don't intersect but just barely touch. For example, we have two segments `[1,2]` and `[2,3]`:
```
·····O·····1=====2·····+·····+·····+·····> x

·····O·····+·····2=====3·····+·····+·····> x
```
We can see that these two segments only have one point in common. Subtracting their endpoints from each other gives:
```
1 - 2 = -1
1 - 3 = -2
2 - 2 =  0
2 - 3 = -1
```
And the resulting difference looks like:
```
·····+····-2====-1=====O·····+·····+·····+·····> x
```
Notice, that in case of only one point in common the Origin is not in the middle of the resulting simplex, but is actually one of its endpoints. When your resulting segment has the Origin at one of its endpoints that means that your original shapes don't intersect, but merely touch at a single point (two segments have only one common point of intersection).

What GJK really says is: if you're able to build a simplex that contains (includes) the Origin then your shapes have at least one or more points of intersection (occupy same points in space).

Now let's take a look at the picture in 2D. Our 2D-space is now an xy-plane (which is represented by two orthogonal number lines instead of a single number line). Every point in our 2D-space now has two xy-coordinates instead of one number, that is, each point is now a 2D-vector. Suppose we have two basic 2D-shapes – a rectangle `ABCD` intersecting a triangle `EFG` on a plane.
```
                 y

                 ^
                 ·
                 ·
                 +           E-----------F
                 ·            \         / 
                 ·             \       /  
                 +     A--------\-----/--------B
                 ·     |         \   /         |
                 ·     |          \ /          |
                 +     |           G           |
                 ·     |                       |
                 ·     |                       |
                 +     D-----------------------C
                 ·
                 ·
·····+·····+·····O·····+·····+·····+·····+·····+·····> x
                 ·
                 ·
                 +
                 ·
                 ·
                 +
                 ·
                 ·
```
These shapes are represented by the following sets of points (2D-vectors, which are pairs of xy-coordinates):
```
A (1, 3)
B (5, 3)
C (5, 1)
D (1, 1)

E (2, 4)
F (4, 4)
G (3, 2)
```
Now, because we have much more numbers here, the arithmetic becomes a little more involved, but its still very easy – we literally subtract all points of one shape from all points of another shape one by one.
```
A - E = (1 - 2, 3 - 4) = (-1, -1)
A - F = (1 - 4, 3 - 4) = (-2, -1)
A - G = (1 - 3, 3 - 2) = (-2,  1)

B - E = (5 - 2, 3 - 4) = ( 3, -1)
B - F = (5 - 4, 3 - 4) = ( 1, -1)
B - G = (5 - 3, 3 - 2) = ( 2,  1)

C - E = (5 - 2, 1 - 4) = ( 3, -3)
C - F = (5 - 4, 1 - 4) = ( 1, -3)
C - G = (5 - 3, 1 - 2) = ( 2, -1)

D - E = (1 - 2, 1 - 4) = (-1, -3)
D - F = (1 - 4, 1 - 4) = (-3, -3)
D - G = (1 - 3, 1 - 2) = (-2, -1)
```
After plotting all of 12 resulting points in our 2D-space and connecting the *outermost* points with lines, we get the following difference shape:
```
                             y
            
                             ^
                             ·
                             ·
                             +     
                             ·     
                             ·     
                 *-----------------------*-----------*
                |            ·                       /
               /             ·                       |
·····+·····+···|·+·····+·····O·····+·····+·····+····/+·····+·····> x
              /              ·                      |
              |              ·                     /
             /   *     *     +     *     *     *   |
             |               ·                    /
            /                ·                    |
            |                +                   /
           /                 ·                   |
           |                 ·                  /
           *-----------*-----------*-----------*
                             ·
                             ·
                             +
                             ·
                             ·
```

GJK says that if we're able to enclose the Origin within the resulting shape, then two original shapes must have collided.
We immediately see, that this shape actually contains the Origin. Therefore we can visually confirm that our original rectangle `ABCD` indeed intersects our original triangle `EFG`.

Now, remember, in 1D to determine whether the resulting segment contains the Origin we check if a primitive inequality `leftEndpoint < Origin < rightEndpoint` holds. So we could think of it as if the segment *surrounds* the Origin from all sides of 1D space (from both left and right, as there are only 2 relative sides in one dimension on our sketch). In 1D for an object to be able to surround any point (the Origin is the zero point on a number line), that object must itself consist of at least two of its own points (in other words, it must be a segment defined by its two points on a number line). Therefore the 1D-version is trying to build a simplex of two endpoints (a segment on a number line). And then it checks whether the Origin is contained within the resulting segment.

To determine if the Origin is enclosed by the resulting difference shape in 2D the algorithm tries to build a 2D-version of a simplex, that is the simplest possible 2D-shape or figure in two-dimensional space which can contain points inside itself. The simplest possible definition of any area on a plane is, ofcourse, a triangle, a minimal set of three points, which can be combined into a shape in our planar space. So while calculating the difference the GJK algorithm actually builds various triangles from three of all difference points of two original shapes and then checks whether that triangular-simplex-shape actually contains the Origin or not. If not, it tries again, until it finally manages to enclose the Origin with any three points of the resulting difference set. If it is not able to build such a triangle from any three points of the difference set even after checking all combinations of points, then there was no collision of rectangle `ABCD` and triangle `EFG`.

... to be continued soon )

## References (must see)
Most of the info (along with reference implementation) was taken from dyn4j. It has a very thorough explanation of GJK and it is definitely a must visit for those who need to understand the intricacies of the algorithm.

1. http://www.dyn4j.org/2010/04/gjk-gilbert-johnson-keerthi/ GJK description (+ a lot of other useful articles)
2. http://mollyrocket.com/849 Awesome old-school GJK / Minkowski space video
3. https://github.com/wnbittle/dyn4j Quality source code for reference

## P.S. 3D-version coming soon...
![3D-version of GJK in plain C coming soon...](http://s21.postimg.org/da9txc3uv/Screen_Shot_2016_01_13_at_09_13_12.jpg "3D-version of GJK in plain C coming soon...")
