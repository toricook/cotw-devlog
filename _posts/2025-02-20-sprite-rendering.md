Today I'm gonna tackle rendering my player to the screen, as well as some other sprites so that my town looks less boring. For this, I'm going to have to start thinking about my architecture a bit.

# Architecture

Yesterday, I put a lot of code into the game class to handle my tilemap rendering, and my first thought today is that this class is going to get really big and convoluted if I don't come up with a better way to separate concerns. The way I want to handle this is by doing the following:

* A set of interfaces will define behaviors that different objects in my game need to implement. For example, IRenderable will have a Render method, and any object that needs to be rendered to the screen will need to implement this
* Each interface will have a corresponding "system" that implements Update and/or Draw, where it executes the relevant method on each interface
* The Game class will know about all of the systems and call their methods in the Game loop

This is kind of like the [Entity Component System](https://en.wikipedia.org/wiki/Entity_component_system) architecture, but also kind of not, because each "component" is an interface that my "entity" will implement rather than its own class that's associated with the entity. The reason for this is just that the interface approach makes a lot more sense to me as someone who has done a lot of OOP in C#, and I have had varying amounts of success when trying to understand how ECS works. I'm hoping that by taking the more familiar approach, I can start to understand the pros and cons a bit better.

# Rendering

Let's work through an example. Right now, I want to write something that will render a bunch of sprites (including my player) to the screen at some pre-specified locations. I'll start with the aforementioned interface. My render method only needs to take in the SpriteBatch, which is needed for MonoGame's rendering loop. I could have it also take in a location in which to render, but I think I'm going to rely on the classes that implement the interface to deal with that for now.

```
  public interface IRenderable
  {
      public void Render(SpriteBatch spriteBatch);
  }
```

Then I need something to implement this interface, which I will call a Sprite. It needs a reference to the Texture2D and source Rectangle so I know what to actually draw, and (for now) it also needs a location in which to draw it. The spriteBatch.Draw() method also takes in a color, which I'm going to hard-code as white for now, which will mean it will just use the colors on the underlying texture.

```
  public class Sprite : IRenderable
  {
      Texture2D _texture;
      Rectangle _source;

      Rectangle _destination;

      public Sprite(Texture2D texture, Rectangle source, Rectangle destination)
      {
          _texture = texture;
          _source = source;
          _destination = destination;
      }

      public void Render(SpriteBatch spriteBatch)
      {
          spriteBatch.Draw(_texture, _destination, _source, Color.White);
      }
  }
```
Lastly, I need to create my rendering system, which just needs to do something like this: 
```
public class RenderingSystem
{
    List<IRenderable> _renderables;

    public RenderingSystem(List<IRenderable> renderables)
    {
        _renderables = renderables;
    }

    public void Draw(SpriteBatch spriteBatch)
    {
        foreach (var renderable in _renderables)
        {
            renderable.Render(spriteBatch);
        }
    }
}
```
And then in my Game class, I can create my RenderingSystem in the Initialize method...but where am I going to get the list of renderables? Somehow, the RenderingSystem needs to know about every renderable that my game is going to use. Let's actually come back to this in a second and think about how we're actually going to be creating our sprites.

# Sprite Loading

Right now, I have a CSV that represents my town map that is being loaded directly in the Game constructor. At some point, I'll probably need some set of classes that keep track of where in the game my player is, which can contain information about which tilemap to render and also which sprites to put where. For now, let's keep things simple by just loading the sprites that should go in the town in a similar way. To render my sprites, I'll need a few pieces of data.

* The texture that contains this sprite, which is just a filename of some file in my Content folder
* The location of the sprite in the texture, which is two integers (x and y)
* The size of the texture, which is also two integers (I think all of my textures are square so we don't really need 2 dimensions here, but I'll do it that way for flexibility)
* The location at which to render the texture, which is also an x and y
* I can technically also provide a width and height for the rendering, if I want to render the sprite smaller or larger than its true size on the sheet. I'm gonna go ahead and say we AREN'T going to allow this because I can't think of a reason to do it for this game

I'm gonna go ahead and make a quick csv by hand to test this, using random sprites from the sprite sheet I downloaded. I realized during this process that having to calculate everything in pixel coordinates is pretty annoying, so I might want to think of a better way in the future, but this will do for now. After that I wrote some super hacky code to load these in my Game class and then pass them in when I make my rendering system.

![Random sprites in my town map](https://github.com/toricook/cotw-devlog/blob/main/cotw/sprites-in-the-town.PNG)

The problem, of course, is that I don't want to have to load every single sprite my game will ever use right at the initialization phase. Not only would it be horribly inefficent, but my game is also going to have random dungeon generation, random enemies spawning, random loot drops, etc. so I don't even *know* what sprites my game is going to need. For that we need...

# The GameObject and the Singleton

My motivation here is the realization that every system is going to need to access every object in the game that implements the interface it's designed to handle. Every time a new object is created, we want the system to know about that. And ideally, we'll also have the ability to destroy objects so that our systems don't have to run computations on objects that have left our scene. This implies some kind of list that exists in our game that every system will have access to. First, let's create the type that this list will contain, which will need to be a base object that everything else in the game will inherit from. We'll call this a GameObject, and it will be abstract because we never want to actually make an instance of a plain old game object, we want to make game objects that do things. My sprite will be one such object. 

```
 public abstract class GameObject
 {
     public GameObject() { }
 }

public class Sprite : GameObject, IRenderable
{
    public Sprite(Texture2D texture, Rectangle source, Rectangle destination) : base()
    {
        _texture = texture;
        _source = source;
        _destination = destination;
    }
}
```

Now, I need to create this object list. Remember, *all* systems need access to this list. There are two opposing thoughts in my head right now about how to do this.

Option 1: Create an abstract System base class that must be constructed with the object list. Make all systems inherit from this. In the Game class, create the object list at initialization and then create each system and pass in a reference to the list.
Option 2: Make the object list a singleton.

I'm going to go with Option 2 for the moment, for two reasons, 1) my Game class is a huge mess right now and I'm going to need to clean it up soon, so I don't really want to add this in there right this second, and 2) I *think* I hate the singleton pattern but I want to try it out to be sure. Not great reasons, I know, but this is my project and I can do what I want.

The idea with the singleton is that there can only ever be one in the entire game. First, we give it a private constructor so that only the singleton is allowed to instantiate itself. Then we give it a static reference to this instance. Finally, we create a public static method Get() so that other systems can access the instance. We should also make this class Sealed so that nothing else can inherit from it and cause weird issues.

```
public sealed class ObjectStore
{
    private ObjectStore() { } 

    static ObjectStore _instance;
    public static ObjectStore Get()
    {
        if (_instance is null)
        {
            _instance = new ObjectStore();
        }
        return _instance;
    }
}
```
The above is a really common pattern that I've seen in tons of places, but it has a pretty glaring issue -- it's not thread safe! If two or more systems called Get() at the same time, it might be possible for all of them to find that _instance is null, which would then cause the ObjectStore to be instantiated multiple times. After going on a side quest about this for a while ([this article](https://csharpindepth.com/Articles/Singleton) was especially interesting), I decided to just go with the incredibly simple option of:
```
  public sealed class ObjectStore
  {
      private ObjectStore() { }
      public static readonly ObjectStore Instance = new ObjectStore();
  }
```
which is thread-safe because static constructors in C# only ever get executed once, even if there are multiple threads, thanks to the CLR (runtime). Yay.

Now I just need to make my ObjectStore actually do the thing I want it to do, which is keep track of all the objects in our game. Since I want to be able to remove objects when they aren't in the game anymore, I'm going to use a HashSet, which will allow O(1) removal by value (rather than having to search potentially the entire list to locate the object of interest). I also want to make it thread-safe so I don't need to worry about things like one system continuing to have access to an object that was just removed by another system. I'll use the ReaderWriterLockSlim, which will give us better performance than just using the lock keyword, because this class will allow multiple systems to read at the same time (which is fine), as long as something isn't currently writing.
```
  public sealed class ObjectStore
  {
      public static readonly ObjectStore Instance = new ObjectStore();

      private readonly HashSet<GameObject> _objects;
      private readonly ReaderWriterLockSlim _lock = new ReaderWriterLockSlim();


      private ObjectStore()
      {
          _objects = new HashSet<GameObject>();
      }

      public void AddObject(GameObject obj)
      {
          _lock.EnterWriteLock();
          try
          {
              _objects.Add(obj);
          }
          finally
          {
              _lock.ExitWriteLock();
          }
      }

      public void RemoveObject(GameObject obj)
      {
          _lock.EnterWriteLock();
          try
          {
              _objects.Remove(obj);
          }
          finally
          {
              _lock.ExitWriteLock();
          }
      }

      public IReadOnlyCollection<GameObject> GetObjects()
      {
          _lock.EnterReadLock();
          try
          {
              return _objects.ToList().AsReadOnly();
          }
          finally
          {
              _lock.ExitReadLock();
          }
      }
  }
```
Lastly, we need to make the GameObjects add and remove themselves from the ObjectStore. Basically we want to do something like this:
```
public abstract class GameObject
{
    public GameObject()
    {
        ObjectStore.Instance.AddObject(this);
    }

    public void Destroy()
    {
        ObjectStore.Instance.RemoveObject(this);
    }
}
```
An immediate downside I notice here is that the classes that inherit from GameObject aren't forced to call this constructor. If I forget to call it, this object will never get added and it'll be a huge pain to understand why. This is getting long though, and I actually want to start testing this thing, so I'm going to add it to my TODO list and move on.

# RenderingSystem Again

*Finally* we can go back to the RenderingSystem and make it use the ObjectStore. I'm going to remove the list of IRenderables and simply write the class as:
```
 public class RenderingSystem
 {
     public void Draw(SpriteBatch spriteBatch)
     {
         foreach (var gameObject in ObjectStore.Instance.GetObjects())
         {
             if (gameObject is IRenderable renderable)
             {
                 renderable.Render(spriteBatch);
             }
         }
     }
 }
```
The check of whether the GameObject implements IRenderable is fast because we know this at compile time, so we're just checking the metadata (not using reflection). So I'm willing to overlook how wasteful this seems for now (yes, we are calling this at 60 fps and yes, we could do better if we just kept track of what is/isn't renderable rather than checking every time). Mainly I just don't want to write more code than I have to, and there aren't going to be a lot of objects in this game. But it is starting to be more clear why more traditional ECS would be better.

ANYWAY, I can now change my hacky sprite loading code to simply create the sprites and not do anything with them, and the act of creation will add them to the ObjectList where the Renderer can find them. By making these changes, I can confirm that this is indeed the case. WOOHOO!!

![Random sprites in my town map](https://github.com/toricook/cotw-devlog/blob/main/cotw/sprites-in-the-town.PNG)

# Sprite Loading Revisited

After all this work, I still have barely done anything to improve the appearance of my little town, which was the whole point. So I am going to revisit my sprite loading code and try to make it more reusable and clean before I get the real sprites in there. As I wander around the town and mines in Castle of the Winds, I notice that each little area has a name, and the one I just drew is called "A Tiny Hamlet". I'm going to call these areas "Scenes". A Scene will have:

* A Tilemap, which is the thing we created last time -- A list of tiles and positions
* A list of sprites and positions that should be in the scene

We'll probably want some more stuff related to the scene, but for now the scene is just a way for us to keep track of which sprite file goes with which tilemap. The scene will have a Load() function that will call the function that parses the Tilemap from a CSV, and then it will parse the sprite CSV. I'll initialize the scene in the Game constructor by hardcoding the paths to the two CSVs (a hack for now). 

Once that class is done, I want to put in the actual sprites that are in that town. Most of them aren't in my spritsheet, sadly, so I had to take some screenshots from the game and make my own.

![Random sprites in my town map](https://github.com/toricook/cotw-devlog/blob/main/cotw/spritesheet.PNG)
(I realized after this that the sign was actually in the sprite sheet. Damn).

Then I did some painstaking counting to figure out where my sprites need to go. At this point, I also realized that I need to apply the same camera offset that I did to the tilemap, or everything will be messed up. I may handle this differently in the future but for now I'm just going to add a Vector2 input to the Render() method on the IRenderables and pass that in to the RenderingSystem. Figuring out where exactly to place each sprite, by the way, was a HUGE pain in the ass. I probably should have drawn these onto my tilemap instead, but alas. At least this sprite rendering code will help with our future, moveable sprites, hopefully. GAH.

![Random sprites in my town map](https://github.com/toricook/cotw-devlog/blob/main/cotw/town-with-stuff.gif)

A couple notes
* The little farm sprite next to the houses at the top isn't quite right. Also made me realize that the colors I got from screenshotting the game are not exactly the same as the ones from my sheet. Will file this away under general sprite maintenance that I need to do later.
* I left out that little hay bucket because I can't find it. Whatever.
* That well in the middle has slime on it in the sprite sheet, and there isn't a normal one. Pretty sure this is a SPOILER for later in the game, but anyway, whatevs.

