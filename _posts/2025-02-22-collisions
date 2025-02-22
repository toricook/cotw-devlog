So my player is able to run all over the tilemap and into the void with impunity, which is not good. It's time to handle collision logic. I'm going to start with making sure he can't walk on top of the houses and other sprites. Yesterday I made the decision that all GameObjects should have a position. Today, I think I need to go one step further and say that they should all not only have a position but a size, or a boundary. This doesn't mean that the player should be unable to collide with any GameObject. The player can, for example, walk on top of loot drops, but he can only pick up the loot if he's standing on top of it. So I still need to know when the player is colliding with this loot. Hence, every GameObject will now have Bounds.
```
public abstract class GameObject
{
    public Point Position { get; set; }
    public int Width { get; set; }
    public int Height { get; set; }
    public Rectangle Bounds
    {
        get
        {
            return new Rectangle(Position, new Point(Width, Height));
        }
    }
    public GameObject(Point position, int width, int height)
    {
        Position = position;
        Width = width; 
        Height = height;
        ObjectStore.Instance.AddObject(this);
    }

    public void Destroy()
    {
        ObjectStore.Instance.RemoveObject(this);
    }

}
```
I also changed every Vector2 to a Point, which uses integers instead of floats, since I'm always using integer positions for everything.

Then I'll make an interface that is implemented by objects that collide. Basically, two objects that implement this interface can never occupy the same space. For now, it's just going to be an empty interface called ICollideable.

The collision checking has to hapen in the movement system because we don't want to allow an object to move to a new position if 1) it implements ICollideable and 2) it would be intersecting another object that implements ICollideable. A brute force way to do this would be to, upon finding that a moveable object is collideable, iterate through every other game object *again* and check if it would collide with this moveable at its new position.
```
  public class MovementSystem
  {
      public void Update(float deltaTime)
      {
          foreach (var gameObject in ObjectStore.Instance.GetObjects())
          {
              if (gameObject is IMoveable moveable)
              {
                  var pos = moveable.GetNextPosition(deltaTime);
                  if (gameObject is ICollideable collideable)
                  {
                      if (HasCollision(gameObject, pos))
                      {
                          return;
                      }
                  }
                  gameObject.Position = pos;
              }
          }
      }

      bool HasCollision(GameObject obj, Point newPosition)
      {
          var newRect = new Rectangle(newPosition, new Point(obj.Width, obj.Height));
          foreach (var gameObject in ObjectStore.Instance.GetObjects())
          {
              if (gameObject == obj) { continue; }
              if (gameObject is ICollideable collideable)
              {
                  if (newRect.X < gameObject.Position.X + gameObject.Width &&
                      newRect.X + newRect.Width > gameObject.Position.X &&
                      newRect.Y < gameObject.Position.Y + gameObject.Height &&
                      newRect.Y + newRect.Height > gameObject.Position.Y)
                  {
                      return true;
                  }
              }
          }
          return false;
      }
  }
```
If I make the sprite class implement ICollideable, this does work, but the collision areas are too big and my character can't get close enough to each building for it to feel natural. This is especially bad in my building sprites because of all that area around each building that is in the way. I think it's time for me to take another approach for these buildings...
