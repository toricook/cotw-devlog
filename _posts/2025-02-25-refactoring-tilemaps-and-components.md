Over the last few days, I did a lot of tedious work to get myself to basically the exact same place I was in already, though in a way that I think will make stuff much easier going forward. Such is the life of a software engineer.

## Art

First, I found a collection of Castle of the Winds sprites from what appears to be an [abandoned project to remake COTW in Elm](https://github.com/mordrax/cotwelm). Thank you mordrax for your service! There's some color imperfections but I'm not going to worry about that now. I think I mentioned this in an earlier post, but I am not copying the tilemaps exactly because I'm too lazy. Might go back and fix later, might not. We'll see.

## Tiled

My Tiled-csv-parsing implementation was not robust enough, so I switched to a library called [DotTiled](https://github.com/dcronqvist/DotTiled) for parsing the .tsx files. I had to rewrite the Tilemap class to turn a DotTiled "Map" object into tile information that can be rendered, and then changed my Tilemap Renderer a bit too. I *also* made two "ObjectLayers" in tiled which contain rectangles (not rendered). One is the Collisions layer and one is the Trigger layer. Collisions will represent areas that players (and enemies, etc.) should not be able to enter, and the triggers will be used for things like entering shops. More on that later. My new map looks like this: 

[MAP PIC]

I added parsing this collision layer to the method in the Tilemap class that parses the data that DotTiled gives me. I also added a way to optionally draw the bounds of each collider using [MonoGame.Primitives2D](https://www.nuget.org/packages/MonoGame.Primitives2D/).

[Collider outline pic]

## Revisiting Components

When I began using the collider, I realized that I needed to make my implementation of the ECS framework more correct or I was going to end up with a huge mess of inheritance which is exactly what ECS tries to avoid.

