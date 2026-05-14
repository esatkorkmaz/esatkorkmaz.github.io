---
title: 'Game Dev Fundamentals Series #3: Ingame Entities and Components'
author: Esat Korkmaz
date: 2026-05-13
category: gamedev-fundamentals
layout: post
---

One of the ways of how modern game engines handle the ingame entity management is based on a object-oriented approach. In Unity (not including DOTS), there's a composition over inheritance approach. All entities are called "GameObject" and they tend to have "components" inherit from MonoBehaviour class. In Unreal, there's a hybrid architecture. All entities are called "Actor" but, for example, entities controlled by player or AI are called "Pawn" which inherit from Actor class. But it's possible to attach a component for adding functionality and behavior to your Actors. Godot engine also adopts a hybrid approach that every entity is called "Node". Nodes can inherit from other Nodes, and also it's possible for Nodes to have Node children in a hierarchical structure. Nodes can serve both a "game object" and "component" purpose. In this way, you can still create composition-based architecture.

In my project, I'm going to embrace Unity's approach. Not because it's "the best" or something like that, all of the three ways have their own use case. It's basically a personal preference. Our project is going to be simple asf, so we're not going to need Unreal's inheritance based approach and our ingame entities can inherit from one class. And Godot's "Everything is a Node" approach might be confusing at the beginning. It's not a hard thing to deal with and it's actually smart, but I think Unity does have a "middle-ground" position among the others. It's not as open ended like Godot, so you know the difference between a "game object" and a "component", but also not quite specific like Unreal. I aim to practice the basics so it's important that we stay organized and be flexible at the same time. As I said, this is mostly a personal preference and you shouldn't struggle much when switching from one to other. Let's get started.

There's no need to overthink our class names. I just called them GameObject and Component like Unity.

```cs
using System;
using Microsoft.Xna.Framework;

public abstract class Component
{
    public string Id { get; protected set; }
    public bool IsEnabled
    {
        get => _isEnabled;

        set
        {
            if (value && !_isEnabled)
            {
                _isEnabled = true;
                OnEnable();
            }
            else if (!value && _isEnabled)
            {
                _isEnabled = false;
                OnDisable();
            }
        }
    }

    public GameObject GameObject { get; private set; }

    // Default ID for components if none is provided
    public const string DEFAULT_ID = "component";

    protected bool _isEnabled = true;
    protected bool _isDestroyed = false;

    public virtual void Initialize(string id, GameObject gameObject)
    {
        Id = id;
        GameObject = gameObject;
    }
    public virtual void LoadContent() { if (!IsValid()) return; }
    public virtual void UnloadContent() { if (!IsValid()) return; }
    public virtual void Update(GameTime gameTime) { if (!IsValid() || !IsEnabled) return; }
    public virtual void Draw(GameTime gameTime) { if (!IsValid() || !IsEnabled) return; }
    public virtual void Destroy() { OnDestroy(); }
    protected virtual void OnEnable() { }
    protected virtual void OnDisable() { }
    protected virtual void OnDestroy()
    {
        _isDestroyed = true;

        Id = null;
        GameObject = null;
    }

    public virtual bool IsValid()
    {
        if (_isDestroyed || GameObject == null)
        {
            throw new Exception($"The component you are trying to access is not valid anymore. It has either been destroyed or it's GameObject reference is null.");
        }

        return true;
    }
}
```

This is our base Component class. All of our components will inherit from this class. 

We do have an id property to avoid any issue if there's more than a single component with the same type in our game objects. Giving an id is optional, and all components will have the id "component" by default if there's no any override parameter. Components having the same id with another component from their type is not allowed. Components of different types can have the same id.

Then we have IsEnabled getter and setters. You may want to execute some actions when a component is enabled or disabled by overriding OnEnable and OnDisable functions.

We have our GameObject, is enabled and is destroyed properties.

There's also the Initialize function. I couldn't use constructor because of how I implemented adding components to GameObjects, which we'll see in our GameObject class.

LoadContent, UnloadContent, Update and Draw functions are called from the parent GameObjects.

Lastly, there's our Destroy and OnDestroy function. Note that the former is for destroying a component, and the latter is for destroy logic.

> ##### WARNING
>
> Because of C#'s garbage collection system, even if a component is 
> destroyed, any reference to the component will prevent it being
> deleted from the memory. Make sure that a component reference you're trying
> to access is valid, if unsure use the IsValid function.
{: .block-warning }

And here's our GameObject code:

```cs
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.Xna.Framework;

public abstract class GameObject
{
    public List<Component> Components { get; private set; } = new List<Component>();

    protected bool _isDestroyed = false;

    public GameObject() { }

    public virtual void LoadContent()
    {
        if (!IsValid()) return;

        foreach (var component in Components)
            component.LoadContent();
    }
    public virtual void UnloadContent()
    {
        if (!IsValid()) return;

        foreach (var component in Components)
            component.UnloadContent();
    }
    public virtual void Update(GameTime gameTime)
    {
        if (!IsValid()) return;
        
        foreach (var component in Components)
            component.Update(gameTime);
    }
    public virtual void Draw(GameTime gameTime)
    {
        if (!IsValid()) return;
        
        foreach (var component in Components)
            component.Draw(gameTime);
    }
    public virtual void Destroy()
    {
        OnDestroy();
    }
    protected virtual void OnDestroy()
    {
        foreach (var component in Components)
            component.Destroy();

        Components.Clear();

        _isDestroyed = true;
    }

    public T AddComponent<T>(string id = Component.DEFAULT_ID) where T : Component, new()
    {
        if (HasComponent<T>(id))
        {
            throw new Exception($"GameObject already has a component {typeof(T).Name} with ID '{id}'");
        }

        var component = new T();

        Components.Add(component);

        component.Initialize(id, this);

        return component;
    }

    public bool HasComponent<T>(string id = Component.DEFAULT_ID) where T : Component
    {
        return Components.OfType<T>().Any(c => c.Id == id);
    }

    public T TryGetComponent<T>(string id = Component.DEFAULT_ID) where T : Component
    {
        return Components.OfType<T>().FirstOrDefault(c => c.Id == id);
    }

    public T GetComponent<T>(string id = Component.DEFAULT_ID) where T : Component
    {
        return Components.OfType<T>().First(c => c.Id == id);
    }

    public void RemoveComponent<T>(string id = Component.DEFAULT_ID) where T : Component
    {
        var component = GetComponent<T>(id);

        component.Destroy();
        Components.Remove(component);
    }

    public T[] GetComponents<T>() where T : Component
    {
        return Components.OfType<T>().ToArray();
    }

    public virtual bool IsValid()
    {
        if (_isDestroyed)
        {
            throw new Exception($"The game object you are trying to access is not valid anymore.");
        }

        return true;
    }
}
```

One thing I want to note that we can't enable or disable game objects like Unity does. When I worked with Unity, disabling game objects caused problems such as not being recognized from other game objects, or problems related to the situation that when we do have a reference to them in our code, but couldn't perform actions because they were disabled. As I know, both Unreal and Godot does not have a feature that disabling the ingame entities entirely. If we would need such an action similar to disabling but not destroying, we can still disable the components that presenting the existence of our game object such as components interact with other game objects or components, or things related to visuals and physics.

> ##### WARNING
>
> Functions related to components such as GetComponent, does not check
> if a relevant component is enabled. If you ever try to access a reference
> from these functions, you might want to check if they're enabled.
{: .block-warning }

Now let's see the examples of how I implemented them, here's the example components:

```cs
using Microsoft.Xna.Framework;

public class TransformComponent : Component
{
    public Vector2 Position { get; set; }
}
```

This is the Transform Component. I'm going to use this component just to store the position value. I didn't override anything else.

```cs
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;

public class SpriteRendererComponent : Component
{
    public Texture2D Texture { get; set; }
    private TransformComponent _transform;

    public override void Initialize(string id, GameObject gameObject)
    {
        base.Initialize(id, gameObject);

        _transform = gameObject.GetComponent<TransformComponent>();
    }

    public override void LoadContent()
    {
        if (Texture == null)
        {
            Texture = Globals.Content.Load<Texture2D>("PNG/Aliens/alienGreen_square");
        }
    }

    public override void Draw(GameTime gameTime)
    {
        base.Draw(gameTime);

        if (Texture != null && _transform != null)
        {
            Globals.SpriteBatch.Draw(Texture, _transform.Position, Color.White);
        }
    }
}
```

This is the Sprite Renderer Component. I'm going to use this component to render my sprites easily. Right now I just load a specific texture only for that game object. This will probably miss some performance benefits if more than one game object use the same content. Loading a content once, and assigning it to the game objects is probably going to be better in that case. We might handle this in the future if needed. Should note that I also created a static class named Globals in Game1 file that has the ContentManager and SpriteBatch properties. This makes the access so easier, we don't have to pass them as parameters or anything like that.

```cs
public static class Globals
{
    public static ContentManager Content { get; set; }
    public static SpriteBatch SpriteBatch { get; set; }
}
```

And here is our example game object:

```cs
using Microsoft.Xna.Framework;

public class ExampleGameObject : GameObject
{
    private TransformComponent _transform;

    public ExampleGameObject()
    {
        _transform = AddComponent<TransformComponent>();

        AddComponent<SpriteRendererComponent>();

        _transform.Position = new Vector2(200, 200);
    }

    public override void LoadContent()
    {
        base.LoadContent();
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

And that's it. Right now we do have an example game object with some components. Notice that unlike Unity, the Transform Component is optional. Not all game objects need to have a Transform Component. For instance, if you have a game object that serves as a game manager, it probably doesn't need to have a position in the space. So this is why it's optional.

Let's add our game object to our test scene.

```cs
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;

public class TestScene : IScene
{
    private ExampleGameObject _testObject;
    private SpriteFont _font;
    private string _text;
    private Vector2 _textPosition;

    public void Initialize()
    {
        _testObject = new ExampleGameObject();
        
        _font = Globals.Content.Load<SpriteFont>("TextFont");
        _text = "Hello, MonoGame!";
        _textPosition = new Vector2(100, 100);
    }

    public void LoadContent()
    {
        _testObject.LoadContent();
    }

    public void UnloadContent()
    {
        _font = null;
        _text = null;

        _testObject.UnloadContent();

        _testObject.Destroy();

        Globals.Content.Unload();
    }

    public void Update(GameTime gameTime)
    {
        _testObject.Update(gameTime);
    }

    public void Draw(GameTime gameTime)
    {
        Globals.SpriteBatch.DrawString(_font, _text, _textPosition, Color.White);

        _testObject.Draw(gameTime);
    }
}
```

And finally, our Game1 class:

```cs
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Content;
using Microsoft.Xna.Framework.Graphics;
using Microsoft.Xna.Framework.Input;

public static class Globals
{
    public static ContentManager Content { get; set; }
    public static SpriteBatch SpriteBatch { get; set; }
}

namespace MonoGame_Project
{
    public class Game1 : Game
    {
        private GraphicsDeviceManager _graphics;
        private SceneManager _sceneManager;

        public Game1()
        {
            _graphics = new GraphicsDeviceManager(this);

            Globals.Content = Content;
            Globals.Content.RootDirectory = "Content";

            IsMouseVisible = true;
        }

        protected override void Initialize()
        {
            _sceneManager = new SceneManager();

            base.Initialize();
        }

        protected override void LoadContent()
        {
            Globals.SpriteBatch = new SpriteBatch(GraphicsDevice);

            _sceneManager.ChangeScene(new TestScene());
        }

        protected override void Update(GameTime gameTime)
        {
            if (GamePad.GetState(PlayerIndex.One).Buttons.Back == ButtonState.Pressed || Keyboard.GetState().IsKeyDown(Keys.Escape))
                Exit();

            _sceneManager.Update(gameTime);

            base.Update(gameTime);
        }

        protected override void Draw(GameTime gameTime)
        {
            GraphicsDevice.Clear(Color.CornflowerBlue);

            Globals.SpriteBatch.Begin();

            _sceneManager.Draw(gameTime);

            Globals.SpriteBatch.End();

            base.Draw(gameTime);
        }
    }
}
```

As you noticed, we have some sort of a hierarchical structure. Components are in GameObjects, and GameObjects are in the scenes, and the scenes are in our Game1 class. Scenes don't access to Components, or Game1 class doesn't access to GameObjects. If we consider it as a tree, in this structure, the nodes only access to their children or the nodes that on the same layer (except Game1 since it's the root) with them.

Let's see the result:

![](/assets/ingame-entities-components.jpg)

In the next part, we're going to look at input management.