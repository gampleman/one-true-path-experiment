# One True Path 

The aim is to create one package that makes working with SVG paths nice and convenient.  
Eventually, this package should replace (part of) the current separate implementations of SVG paths in existing packages.

*I'll refer to the package here as `OneTruePath`, but we might want to pick a more serious name later.*

### in scope 

* a data structure for representing SVG draw instructions
* a pretty-printer (convert draw instructions to strings)
* a parser (convert strings to draw instructions)  - could be a separate package
* basic transformations:
    For instance translating a set of instructions by some amount, get the bounding box of a path, 
    visually concatenate paths, overlay paths.  

### out of scope (for now) 

* advanced mathematics: 
    Stuff like derivatives, tangents, interpolation is probably best implemented on top of OneTruePath. Maybe there are good practical reasons to implement mathematical funcions directly on svg instructions. 
    We'll have to investigate what exactly makes sense to include here and what truely belongs in a separate package.

* different backends:
    - Canvas is compatable with svg (has move, arc and cubicBezier, so all svg instructions can be mapped to canvas ones), but does not 
    have adequate support in elm right now
    - WebGL would involve writing a rasterizer (convert curves to pixels)

## SVG Path usage in current packages

Based on conversations at elm europe and searching package.elm-lang.org, these packages use svg paths: 

* [terezka/elm-plot](http://package.elm-lang.org/packages/terezka/elm-plot/latest)
* [gampleman/elm-visualization](http://package.elm-lang.org/packages/gampleman/elm-visualization/latest)
* [mdgriffith/elm-style-animation](http://package.elm-lang.org/packages/mdgriffith/elm-style-animation/latest)
* [opensolid/svg](http://package.elm-lang.org/packages/opensolid/svg/latest)
* [folkertdev/svg-path-dsl](http://package.elm-lang.org/packages/folkertdev/svg-path-dsl/latest)

Here are some observations and links to relevant types and functions 

### elm-plot

Only uses the absolute commands (relative ones are not expressable). Its `Command` type can be found [here](https://github.com/terezka/elm-plot/blob/master/src/Internal/Draw.elm) 
The same file also defines a function that maps over all the coordinates in the structure [(source)](https://github.com/terezka/elm-plot/blob/master/src/Internal/Draw.elm#L279) which can 
be generalized to `(Coordinate -> Coordinate) -> Path -> Path`. 

### elm-visualization

Only supports a subset of the path instructions, and adds some new ones (ArcCustom, Rect). 
The main data structure is [PathSegment](https://github.com/gampleman/elm-visualization/blob/79ce8ecf7d208a2969805085a64b1017dce5334d/src/Visualization/Path.elm#L47) .

The custom [ArcCustom](https://github.com/gampleman/elm-visualization/blob/79ce8ecf7d208a2969805085a64b1017dce5334d/src/Visualization/Path.elm#L142) instruction operates with a startAngle and endAngle, something that is also defined by elm-style-animation. This is not part of the SVG standard, but maybe useful enough in practice to include 
as a first-class instruction?

### svg-path-dsl


This package contains separate instructions for commands with one argument `L0,0` and with multiple arguments `L1,1 2,3`. With a fresh view, having one instruction is obviously better. 
Otherwise, it is pretty similar to the Path type and its api proposed here.  

### elm-style-animation

elm-style-animation has a [PathCommand](https://github.com/mdgriffith/elm-style-animation/blob/86f81b0f5a28289894fe61c14fa2c34c0bf895ec/src/Animation/Model.elm#L97) type
that is rendered by [ pathCmdValue](https://github.com/mdgriffith/elm-style-animation/blob/197f23a6daea8eee3337d54edf7da4570710ea8b/src/Animation.elm#L2141). It looks like
one constructor of the PathCommand type can construct multiple commands.

### opensolid/svg

the opensolid/svg package uses polygon and polyline where possible, and draws arcs and curves using manual string concatenation.
It also defines curves and polylines in a more abstract way, and implements many mathematical operations on them.

## My Experiment 

This is an experiment to see what kind of API and data structure is convenient. Feedback is welcome.

There are currently 4 modules  

* Path 
    A path is a list of subpaths. A subpath is one moveto instruction (relative or absolute) followed by a list of drawto instructions (drawto includes all the other
    instructions, like `L`, `Q`, etc.)

    The main data strucures look like 

    ```elm
    type alias Path =
        List SubPath

    type alias SubPath =
        { moveto : MoveTo, drawtos : List DrawTo }

    type MoveTo
        = MoveTo Mode Coordinate

    type DrawTo
        = LineTo Mode (List Coordinate)
        | Horizontal Mode (List Float)
        | Vertical Mode (List Float)
        | CurveTo Mode (List ( Coordinate, Coordinate, Coordinate ))
        | SmoothCurveTo Mode (List ( Coordinate, Coordinate ))
        | QuadraticBezierCurveTo Mode (List ( Coordinate, Coordinate ))
        | SmoothQuadraticBezierCurveTo Mode (List Coordinate)
        | EllipticArc Mode (List EllipticalArcArgument)
        | ClosePath
    
    type Mode = Absolute | Relative 
    ```

    With the current api, we can write things like 
    
    ```elm
    myPath :: Path
    myPath =
        [ subpath (moveTo ( 0, 0 ))
            [ lineTo [ ( 100, 0 ), ( 100, 100 ), ( 0, 100 ) ]
            , closepath
            ]
        , subpath (moveBy ( 42, 42 ))
            [ lineBy [ ( 30, 20 ) ]
            ]
        ]
    ```

    This module also includes stringify functions to convert to something that the `d` attribute (and the parser) accepts.
* Command
    Path makes a distinction (in the types) between `MoveTo` and `DrawTo`. In general that is great, but sometimes it is nicer to be able to easily compose 
    movetos and drawtos. Command is a wrapper for that, with conversion functions to and from Path.
* PathParser
    Parse some svg path syntax into an elm `Path` object.


And then some experiments 

* ToAbsolute 
    convert any path to a path consisting solely of absolute instructions. Looking for ways to make this code nice
    
    The main problem here is threading the CursorState record through the list of instructions. I've used my 
    [elm-state](http://package.elm-lang.org/packages/folkertdev/elm-state/2.1.0/) package to make this more convenient. 
    (yup, it's the state monad, but this is a case where I genuinly think it makes code nicer, which is not often in elm)

    Then there remains a function that describes what every instruction does to the DrawState, and that's just going to be 
    a long function that will be tedious to write.

* PlayGround
    some random experiments/tests

## Current packages and this proposal 

* elm-style-animation

    This module uses a custom type [at the nodes of its PathCommand type](https://github.com/mdgriffith/elm-style-animation/blob/86f81b0f5a28289894fe61c14fa2c34c0bf895ec/src/Animation/Model.elm#L97) (not `(Float, Float)` but something more elaborate). It probably makes most sense to 
    create a function `Model.PathCommand -> OneTruePath.Command`, then use the stringification in the shared package.

    Alternatively, the elm-style-animation package could be changed to work more naturally with (sub)paths (i.e. enforcing a leading MoveTo)
    

* elm-plot 
    Looks like it can use `OneTruePath.Path` and its api
    
* elm-visualization 
    Looks like it can use `OneTruePath.Path` and its api
    
* opensolid/svg
    Looks like it can use `OneTruePath.Path` and its api

* svg-path-dsl

    will be deprecated when OneTruePath is released 

