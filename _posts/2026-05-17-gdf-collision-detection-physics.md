---
title: 'Game Dev Fundamentals Series #5: Collision Detection and Physics'
author: Esat Korkmaz
date: 2026-05-17
category: gamedev-fundamentals
layout: post
---

As I mentioned this series is not about engine programming, so we're not going write our fully functional physics engine or collision detection code from scratch. We're going to implement the most obvious and simple method of collision detection which is the AABB (Axis-Aligned Bounding Box) algorithm. You simply use inequality checks to see if the extents of two rectangular shapes overlap on both the X and Y axes.

First of all, I want to mention that I created a helper class named DebugDraw to see our colliders in the screen:

```cs
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;

public static class DebugDraw
{
    static Texture2D _pixel;

    public static void Init(GraphicsDevice gd)
    {
        _pixel = new Texture2D(gd, 1, 1);
        _pixel.SetData([Color.White]);
    }

    public static void Rect(SpriteBatch sb, Rectangle r, Color c)
    {
        sb.Draw(_pixel, new Rectangle(r.X, r.Y, r.Width, 1), c);
        sb.Draw(_pixel, new Rectangle(r.X, r.Bottom, r.Width, 1), c);
        sb.Draw(_pixel, new Rectangle(r.X, r.Y, 1, r.Height), c);
        sb.Draw(_pixel, new Rectangle(r.Right, r.Y, 1, r.Height), c);
    }
}
```

Our game objects are going to have this collision component:
 
```cs
using Microsoft.Xna.Framework;

public class CollisionComponent : Component
{
    public Rectangle Bounds { get; private set; }
    public int Width { get; set; }
    public int Height { get; set; }

    private TransformComponent _transform;

    public override void Initialize(string id, GameObject gameObject)
    {
        base.Initialize(id, gameObject);

        _transform = gameObject.GetComponent<TransformComponent>();
    }

    public void RefreshBounds()
    {
        Bounds = new Rectangle(
            (int)_transform.Position.X,
            (int)_transform.Position.Y,
            Width,
            Height
        );
    }

    public override void Draw(GameTime gameTime)
    {
        base.Draw(gameTime);

        DebugDraw.Rect(Globals.SpriteBatch, Bounds, Color.Red);
    }
}
```

These bounds have the information of the X and Y position, the width and height of our collider, necessary for our collision detection.

Then we have our collision system class. This is going to be a little bit long so let me take it step by step:

```cs
public enum CollisionEventType { Enter, Stay, Exit }

public struct CollisionResult
{
    public GameObject A;
    public GameObject B;
    public CollisionEventType EventType;
}
```

We have our collision event types, and collision results. Two gameobjects collide with each other, and an event occurs. And we're going to store them in our collision system class. Collision results are going to be useful when we implemented our physics system.


```cs
public class CollisionSystem
{
    private HashSet<(GameObject, GameObject)> _previousPairs = new();
    private HashSet<(GameObject, GameObject)> _currentPairs = new();

    public IReadOnlyList<CollisionResult> Results { get; private set; } = new List<CollisionResult>();

    ...
```

We have two pairs because if we want to be able to track the collision exit events, we should compare both pairs from the previous frame and our current frame. There's also our collision results stored.

```cs
    private bool Intersects(GameObject a, GameObject b)
    {
        var colA = a.TryGetComponent<CollisionComponent>();
        var colB = b.TryGetComponent<CollisionComponent>();

        if (colA == null || colB == null) return false;

        return colA.Bounds.Intersects(colB.Bounds);
    }

    ...
```

And here's our function checks if there's an overlap between two gameobjects' bounds. This is simply doing the inequality check I mentioned above.

```cs
    public void Update(IReadOnlyList<GameObject> gameObjects)
    {
        _currentPairs.Clear();

        var collidables = new List<GameObject>();

        foreach (var go in gameObjects)
        {
            var col = go.TryGetComponent<CollisionComponent>();
            if (col != null && col.Width > 0 && col.Height > 0)
                collidables.Add(go);
        }

        for (int i = 0; i < collidables.Count; i++)
        {
            for (int j = i + 1; j < collidables.Count; j++)
            {
                var a = collidables[i];
                var b = collidables[j];

                if (Intersects(a, b))
                {
                    var pair = a.Id.CompareTo(b.Id) < 0 ? (a, b) : (b, a);
                    _currentPairs.Add(pair);
                }
            }
        }

        var results = new List<CollisionResult>();

        foreach (var (a, b) in _currentPairs)
        {
            var eventType = _previousPairs.Contains((a, b))
                ? CollisionEventType.Stay
                : CollisionEventType.Enter;

            results.Add(new CollisionResult { A = a, B = b, EventType = eventType });
        }

        foreach (var (a, b) in _previousPairs)
        {
            if (!_currentPairs.Contains((a, b)))
                results.Add(new CollisionResult { A = a, B = b, EventType = CollisionEventType.Exit });
        }

        Results = results;
        _previousPairs = [.. _currentPairs];
    }
}
```

Then, in our Update function, we get all the game objects with CollisionComponent. Each of the game objects check for a potential overlap between the other game objects with the same component, and if there's an intersection, they're going to be added to the current pair hashset, which is going to be cleaned in the next frame. We also check if a collision happened just now or still happening. In this way, it is possible to separate collision enter and collision stay events from each other.

Our current collision detection code is not good for performance since all of the game objects check every other game objects. This is not a major issue in our case since we only have two game objects in our scene. However, as more game objects are added, performance will drop significantly.

One of the solutions for this problem is a spatial partitioning algorithm. I'm not going to get into details, but I write an algorithm related to this topic in raylib, using C++. [Here's](https://github.com/esatkorkmaz/raylib-spatial-hashing) the related GitHub repository. However, as I said this is not a big deal right now and we keep it the "brute force" way.

Afterwards, all the collision results of the current pairs are being stored. We also do check for any collision exit event, if there's a pair exists in the pairs from the previous frame, but not among the current frame pairs.

So this is roughly a collision system. Now I want to implement some physical stuff communicate with it.

Here's a physics based component for our alien game object:

```cs
using Microsoft.Xna.Framework;

public class PhysicsComponent : Component
{
    public Vector2 Velocity { get; set; }
    public bool IsGrounded { get; private set; }

    private const float Gravity = 800f;

    private TransformComponent _transform;
    private CollisionComponent _collision;

    public override void Initialize(string id, GameObject gameObject)
    {
        base.Initialize(id, gameObject);

        _transform = gameObject.GetComponent<TransformComponent>();
        _collision = gameObject.GetComponent<CollisionComponent>();
    }

    public override void Update(GameTime gameTime)
    {
        base.Update(gameTime);

        float dt = (float)gameTime.ElapsedGameTime.TotalSeconds;

        if (!IsGrounded)
            Velocity = new Vector2(Velocity.X, Velocity.Y + Gravity * dt);

        _transform.Position += Velocity * dt;

        IsGrounded = false;
    }

    public void HandleCollision(GameObject other, CollisionEventType eventType)
    {
        if (eventType == CollisionEventType.Exit) return;

        var otherCollider = other.TryGetComponent<CollisionComponent>();
        if (otherCollider == null) return;

        var otherBounds = otherCollider.Bounds;
        var myBounds = _collision.Bounds;

        bool fallingDown = Velocity.Y >= 0;
        bool feetNearTop = myBounds.Bottom >= otherBounds.Top && myBounds.Bottom <= otherBounds.Top + 16;

        if (fallingDown && feetNearTop)
        {
            _transform.Position = new Vector2(
                _transform.Position.X,
                otherBounds.Top - myBounds.Height + 1
            );

            Velocity = new Vector2(Velocity.X, 0);
            IsGrounded = true;
        }
    }
}
```

We have our Update function which is going to make our alien character fall. And there's a function called HandleCollision, which is going to stop our character fall and will mark it as grounded. This is going to be executed from our physics system below:

```cs
using System.Collections.Generic;

public class PhysicsSystem
{
    public void Resolve(IReadOnlyList<CollisionResult> results)
    {
        foreach (var result in results)
        {
            TryResolve(result.A, result.B, result.EventType);
            TryResolve(result.B, result.A, result.EventType);
        }
    }

    private void TryResolve(GameObject subject, GameObject other, CollisionEventType eventType)
    {
        var physics = subject.TryGetComponent<PhysicsComponent>();
        if (physics == null) return;

        physics.HandleCollision(other, eventType);
    }
}
```

We get the collision results from our collision system and resolve all of them.

So let's get to connecting the systems. This is going to be done from our scene:

```cs
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.Xna.Framework;

public abstract class GameScene
{
    private readonly CollisionSystem _collisionSystem = new();
    private readonly PhysicsSystem _physicsSystem = new();
    private readonly List<GameObject> gameObjects = new();

    public IReadOnlyList<GameObject> GameObjects => gameObjects.AsReadOnly();

    public virtual void Initialize() { }

    public virtual void LoadContent()
    {
        foreach (var go in gameObjects)
            go.LoadContent();
    }

    public virtual void UnloadContent()
    {
        foreach (var go in gameObjects)
        {
            go.UnloadContent();
            go.Destroy();
        }

        gameObjects.Clear();
        Globals.Content.Unload();
    }

    public virtual void Update(GameTime gameTime)
    {
        foreach (var go in gameObjects)
            go.Update(gameTime);

        foreach (var go in gameObjects)
            go.TryGetComponent<CollisionComponent>()?.RefreshBounds();

        _collisionSystem.Update(GameObjects);
        _physicsSystem.Resolve(_collisionSystem.Results);
    }

    public virtual void Draw(GameTime gameTime)
    {
        foreach (var go in gameObjects)
            go.Draw(gameTime);
    }

    public void AddGameObject(GameObject gameObject)
    {
        if (HasGameObject(gameObject.Id))
            throw new Exception($"GameObject with ID '{gameObject.Id}' already exists");

        gameObjects.Add(gameObject);
    }

    public bool HasGameObject(string id) => gameObjects.Any(go => go.Id == id);

    public GameObject FindGameObject(string id) => gameObjects.Find(go => go.Id == id);

    public void RemoveGameObject(string id)
    {
        var go = FindGameObject(id);
        if (go == null) return;

        go.UnloadContent();
        go.Destroy();
        gameObjects.Remove(go);
    }
}
```

Now this is where things get tricky. Take a look at the Update function again:

```cs
public virtual void Update(GameTime gameTime)
{
    foreach (var go in gameObjects)
        go.Update(gameTime);

    foreach (var go in gameObjects)
        go.TryGetComponent<CollisionComponent>()?.RefreshBounds();

    _collisionSystem.Update(GameObjects);
    _physicsSystem.Resolve(_collisionSystem.Results);
}
```

I'm going to explain what is going on in here, but I want to show how I added related components to my example game object first:

```cs
using Microsoft.Xna.Framework;

public class ExampleGameObject : GameObject
{
    private TransformComponent _transform;
    private CollisionComponent _collision;
    private SpriteRendererComponent _spriteRenderer;

    public ExampleGameObject(string id) : base(id)
    {
        _transform = AddComponent<TransformComponent>();
        _collision = AddComponent<CollisionComponent>();

        AddComponent<PhysicsComponent>();

        _spriteRenderer = AddComponent<SpriteRendererComponent>();
        _spriteRenderer.SetContentPath("PNG/Aliens/alienGreen_square");

        _transform.Position = new Vector2(200, 200);
    }

    public override void LoadContent()
    {
        base.LoadContent();

        _collision.Width = _spriteRenderer.Texture.Width;
        _collision.Height = _spriteRenderer.Texture.Height;
    }

    public override void Update(GameTime gameTime)
    {
        base.Update(gameTime);
    }

    public override void Draw(GameTime gameTime)
    {
        base.Draw(gameTime);
    }
}
```

As you see, we added these components in order; Transform, collision, physics and sprite renderer.

Component order is important since we update them in a foreach loop. It is based on our components list order.

If you're going to take a look at PhysicsComponent again, you can see that it's depends on the CollisionComponent (and also TransformComponent) since it's trying to get a reference of it first:

```cs
using Microsoft.Xna.Framework;

public class PhysicsComponent : Component
{
    public Vector2 Velocity { get; set; }
    public bool IsGrounded { get; private set; }

    private const float Gravity = 800f;

    private TransformComponent _transform;
    private CollisionComponent _collision;

    public override void Initialize(string id, GameObject gameObject)
    {
        base.Initialize(id, gameObject);

        _transform = gameObject.GetComponent<TransformComponent>();
        _collision = gameObject.GetComponent<CollisionComponent>();
    }

    ...
```

This means you can't add PhysicsComponent before adding CollisionComponent, otherwise the code will throw an exception. So that's why we do vice versa.

But the problem is, PhysicsComponent is updating our transform position AFTER we had our business with CollisionComponent (since it comes before than PhysicsComponent in our foreach loop) and this causes CollisionComponent being unaware of the changes PhysicsComponent made on the transform position since it also depends on TransformComponent. So updating bounds in Update function would be broken.

Modern game engines handle this "execution order dependency" situation in different ways:

In Unity, the game engine provides you a literal Script Execution Order list.

Unreal provides an explicit dependency for every actor and component with "AddTickPrerequisite" function. Unreal also separate ticks to groups.

Godot has a different approach. For minimizing the execution order dependency, it provides a deferred call system, which executes your function until it is safe to do so.

Now if we get back to our TestScene's update loop, this is why we call UpdateBounds functions after updating all of our game objects.

```cs
public virtual void Update(GameTime gameTime)
{
    foreach (var go in gameObjects)
        go.Update(gameTime);

    foreach (var go in gameObjects)
        go.TryGetComponent<CollisionComponent>()?.RefreshBounds();

    _collisionSystem.Update(GameObjects);
    _physicsSystem.Resolve(_collisionSystem.Results);
}
```

We also update our physics system after collision system since it depends on the collision results the collision system will offer.

And we update our systems after game objects because the positions our collision system checks are being changed in our game object components. (PhysicsComponent in this case)

After that, I created a platform game object for test purposes:

```cs
using Microsoft.Xna.Framework;

public class PlatformEntity : GameObject
{
    private TransformComponent _transform;
    private CollisionComponent _collision;
    private SpriteRendererComponent _spriteRenderer;

    public PlatformEntity(string id) : base(id)
    {
        Id = id;

        _transform = AddComponent<TransformComponent>();

        _collision = AddComponent<CollisionComponent>();

        _spriteRenderer = AddComponent<SpriteRendererComponent>();
        _spriteRenderer.SetContentPath("PNG/Wood elements/elementWood012");

        _transform.Position = new Vector2(125, 400);
    }

    public override void LoadContent()
    {
        base.LoadContent();

        _collision.Width = _spriteRenderer.Texture.Width;
        _collision.Height = _spriteRenderer.Texture.Height;
    }

    public override void Update(GameTime gameTime)
    {
        base.Update(gameTime);
    }

    public override void Draw(GameTime gameTime)
    {
        base.Draw(gameTime);
    }
}
```

And finally added my platform game object to the TestScene class:

```cs
using System;
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;

public class TestScene : GameScene
{
    private SpriteFont _font;
    private string _text;
    private Vector2 _textPosition;
    private Action<InputAction> _destroyHandler;

    public override void Initialize()
    {
        AddGameObject(new ExampleGameObject("example"));
        AddGameObject(new PlatformEntity("platform"));

        _font = Globals.Content.Load<SpriteFont>("TextFont");
        _text = "Hello, MonoGame!";
        _textPosition = new Vector2(100, 100);

        _destroyHandler = (action) =>
            {
                RemoveGameObject("example");
            };

        InputManager.Instance.OnDestroyActionPressed += _destroyHandler;
    }

    public override void LoadContent()
    {
        base.LoadContent();
    }

    public override void UnloadContent()
    {
        InputManager.Instance.OnDestroyActionPressed -= _destroyHandler;
        _destroyHandler = null;

        _font = null;
        _text = null;

        base.UnloadContent();
    }

    public override void Update(GameTime gameTime)
    {
        base.Update(gameTime);
    }

    public override void Draw(GameTime gameTime)
    {
        Globals.SpriteBatch.DrawString(_font, _text, _textPosition, Color.White);

        base.Draw(gameTime);
    }
}
```

And here's the result, our character falls to the top of our platform:

![](/assets/collision-detection-physics.jpg)

In the next part, we're going to look at camera systems.