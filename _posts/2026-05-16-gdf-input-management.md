---
title: 'Game Dev Fundamentals Series #4: Input Management'
author: Esat Korkmaz
date: 2026-05-16
category: gamedev-fundamentals
layout: post
---

Are you one of the guys that binding any actions to the keys itself? In your code, do you check the literal Space key to jump? You're most likely in trouble if you ever decide to make it cross-platform, because I don't remember if there's any Space key in a PlayStation console?

So what? Should we get through a painful stage everytime when we try to port our inputs? Of course not! Input management exactly handles this issue. Instead of executing things by checking our inputs directly, we build a bridge between our actions and input keys and they are separate from each other.

In this part, we're going to destroy the cube in our scene. This is going to be an action. And this action is going to be triggered by pressing the Space key in our keyboard. But we're going to do this in an event based input manager. Our input manager is still going to be keyboard based, so it's going to check only for our keyboard state, but it's such flexible that you can adopt to whatever else you want.

This is pretty barebones, and it's definitely not production-ready, but I only want you to get the idea.

```cs
public enum InputAction
{
    DESTROY_THE_CUBE
}
```

This is our input actions enum. Input actions are unique so enums make sense. Could also use strings too. But I can see all enums when I write "InputAction." in my code editor. Also, this is probably better for performance.

```cs
public class InputMap
{
    private Dictionary<InputAction, HashSet<Keys>> _bindings = [];

    public void Register(InputAction action, params Keys[] keys)
    {
        _bindings[action] = [.. keys];
    }

    public void AddKey(InputAction action, Keys key)
    {
        if (!IsRegistered(action)) return;

        _bindings[action].Add(key);
    }

    public void RemoveKey(InputAction action, Keys key)
    {
        _bindings[action].Remove(key);
    }

    public void ClearKeys(InputAction action)
    {
        _bindings[action].Clear();
    }

    public IReadOnlyDictionary<InputAction, HashSet<Keys>> GetAllBindings()
    {
        return _bindings;
    }

    public HashSet<Keys> GetKeys(InputAction action)
    {
        return _bindings[action];
    }

    public bool IsRegistered(InputAction action)
    {
        return _bindings.ContainsKey(action);
    }

    public InputAction? FindActionByKey(Keys key)
    {
        foreach (var binding in _bindings)
        {
            if (binding.Value.Contains(key)) return binding.Key;
        }

        return null;
    }
}
```

This is our input map. Basically this is the "bridge" where all actions and keys stored as dictionaries. An input action can have more than one key. I used HashSet type to prevent duplications in key bindings. Also, the current code allows you to add the same key to more than one action, I haven't tested the consequences of it, probably both bindings would work if you ever try. But if you don't want more than one action uses the same key, there's a function you can use for your purposes called "FindActionByKey" to check if there's any action has the key value you passed.

```cs
public class InputManager
{
    public static InputManager Instance { get; private set; }

    private readonly InputMap _map;
    private KeyboardState _currentState;
    private KeyboardState _previousState;

    public event Action<InputAction> OnDestroyActionPressed;

    public InputManager(InputMap map)
    {
        Instance = this;
        _map = map;
    }

    public void Update()
    {
        _previousState = _currentState;
        _currentState = Keyboard.GetState();

        foreach (var action in _map.GetAllBindings().Keys)
        {
            if (IsActionJustPressed(action))
                OnDestroyActionPressed?.Invoke(action);
        }
    }

    public bool IsActionDown(InputAction action)
    {
        foreach (var key in _map.GetKeys(action))
            if (_currentState.IsKeyDown(key)) return true;

        return false;
    }

    public bool IsActionJustPressed(InputAction action)
    {
        foreach (var key in _map.GetKeys(action))
            if (_currentState.IsKeyDown(key) && _previousState.IsKeyUp(key)) return true;

        return false;
    }

    public bool IsActionJustReleased(InputAction action)
    {
        foreach (var key in _map.GetKeys(action))
            if (_currentState.IsKeyUp(key) && _previousState.IsKeyDown(key)) return true;

        return false;
    }
}
```

Lastly, this is our input manager. Before getting into destroying the cube section, I want to say that I changed the scene interface, now it's an abstract class and manages the ingame entities in a list. This was a must since previously I managed the game object's code directly from the scene and this caused problems when I tried to destroy it. (related to the previous part's invalid component reference stuff) This is also going to be useful in our next part.

```cs
using System;
using System.Collections.Generic;
using System.Linq;
using Microsoft.Xna.Framework;

public abstract class GameScene
{
    private readonly List<GameObject> gameObjects = new List<GameObject>();

    public IReadOnlyList<GameObject> GameObjects { get => gameObjects.AsReadOnly(); }

    public virtual void Initialize() { }
    public virtual void LoadContent()
    {
        foreach (var gameObject in gameObjects)
            gameObject.LoadContent();
    }
    public virtual void UnloadContent()
    {
        foreach (var gameObject in gameObjects)
        {
            gameObject.UnloadContent();
            gameObject.Destroy();
        }

        gameObjects.Clear();

        Globals.Content.Unload();
    }
    public virtual void Update(GameTime gameTime)
    {
        foreach (var gameObject in gameObjects)
            gameObject.Update(gameTime);
    }
    public virtual void Draw(GameTime gameTime)
    {
        foreach (var gameObject in gameObjects)
            gameObject.Draw(gameTime);
    }

    public void AddGameObject(GameObject gameObject)
    {
        if (HasGameObject(gameObject.Id))
        {
            throw new Exception($"GameObject with ID '{gameObject.Id}' already exists");
        }

        gameObjects.Add(gameObject);
    }

    public bool HasGameObject(string id)
    {
        return gameObjects.Any(go => go.Id == id);
    }

    public GameObject FindGameObject(string id)
    {
        return gameObjects.Find(go => go.Id == id);
    }

    public void RemoveGameObject(string id)
    {
        var gameObject = FindGameObject(id);

        if (gameObject != null)
        {
            gameObject.UnloadContent();
            gameObject.Destroy();

            gameObjects.Remove(gameObject);
        }
    }
}
```

Should note that I also added an Id property to the GameObject class to make them unique.

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

And here's the result before pressing the space key, after the scene is loaded:

![](/assets/ingame-entities-components.jpg)

Here's the afterwards:

![](/assets/scene-management.jpg)

In the next part, we're going to look at collision detection and physics.