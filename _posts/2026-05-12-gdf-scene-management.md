---
title: 'Game Dev Fundamentals Series #2: Scene Management'
author: Esat Korkmaz
date: 2026-05-12
category: gamedev-fundamentals
layout: post
---

Video games includes different stages to complete any kind of objective. These stages are called levels. Modern game engines like Unity and Unreal treat levels as seperated stages, but Godot has a different approach that includes both levels and "templates" for game objects (bullets, NPCs etc.) in a single entity. In Unity and Unreal, these "templates" for game objects are called Prefabs and Blueprint Classes in order, making them seperate from levels. In Godot, you do both levels and templates in a single entity called "scenes". (different from Unity's scenes which only meant to be used for levels) To avoid confusion and being simple, I'm also going to seperate the levels and templates.

Let's call our levels as "scenes" like Unity does. As I said, this is different from Godot's "scene" concept. Our scenes are just levels like I mentioned above.

Before getting into scene management, I want to mention about some fundamentals of MonoGame. When you create a new MonoGame project and look at the file called "Game1.cs", this is what you see:

```cs
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Graphics;
using Microsoft.Xna.Framework.Input;

namespace Project1
{
    public class Game1 : Game
    {
        private GraphicsDeviceManager _graphics;
        private SpriteBatch _spriteBatch;

        public Game1()
        {
            _graphics = new GraphicsDeviceManager(this);
            Content.RootDirectory = "Content";
            IsMouseVisible = true;
        }

        protected override void Initialize()
        {
            // TODO: Add your initialization logic here

            base.Initialize();
        }

        protected override void LoadContent()
        {
            _spriteBatch = new SpriteBatch(GraphicsDevice);

            // TODO: use this.Content to load your game content here
        }

        protected override void Update(GameTime gameTime)
        {
            if (GamePad.GetState(PlayerIndex.One).Buttons.Back == ButtonState.Pressed || Keyboard.GetState().IsKeyDown(Keys.Escape))
                Exit();

            // TODO: Add your update logic here

            base.Update(gameTime);
        }

        protected override void Draw(GameTime gameTime)
        {
            GraphicsDevice.Clear(Color.CornflowerBlue);

            // TODO: Add your drawing code here

            base.Draw(gameTime);
        }
    }
}
```

There's lots of things we're gonna dive into it later but basically what matters now is functions called Initialize, LoadContent, Update and Draw.

Initialize is the function that called once the game is started. Useful for initializing logical things like lists, dictionaries, video settings etc., things that relatively more back-end related.

LoadContent is where you load the assets of your game. Called once before the game loop begins.

Update is a function called every frame. Used for input, physics, movement and AI etc. basically things that needs to be checked every moment. There's also a parameter of it called gameTime. This is a crucial variable that stores the time data that independent from your frame to prevent things like physics to be frame-dependent. Ever had an issue that, especially in older games ported to PC or console exclusive ones played with emulators, physics or just the game loop in general be ridiculously faster when you had an higher FPS? This is because they didn't implement this method! Game consoles do have FPS standards and old school programmers tend to write their update logic dependent to that specific FPS value. I don't know if it was just laziness or hardware restrictions. But, after all, MonoGame lets an opportunity for us to prevent these issues!

Lastly, we have a function called Draw that called after every Update call. Every visual thing is handled by this function. After we checked and calculated every necessary thing in our Update function, we draw our visual materials as a result.

So this is basically what MonoGame brings in the beginning. Let's get back into scene management.

First of all, for scene management, obviously, we need to create our scenes. To keep things simple, I'm just going to create a scene interface, then create separate scene classes inherited from this interface.

```cs
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Content;
using Microsoft.Xna.Framework.Graphics;

public interface IScene
{
    void LoadContent(ContentManager content, SpriteBatch spriteBatch);
    void UnloadContent();
    void Update(GameTime gameTime);
    void Draw(GameTime gameTime);
}
```

LoadContent, Update and Draw parameters are going to be passed from our Game1 class. Unlike game engines like Unity, these calls must be handled from our Game1 class. Just writing these functions will not work. This is because we don't have a "game object" concept yet. We're going to address this issue later, now to handle our scene logic, I'm going to create a class to manage our scenes like this:

```cs
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Content;
using Microsoft.Xna.Framework.Graphics;

public class SceneManager
{
    private ContentManager _content;
    private SpriteBatch _spriteBatch;
    private IScene currentScene;

    public SceneManager(ContentManager content)
    {
        _content = content;
    }

    public void InitializeSpriteBatch(SpriteBatch spriteBatch)
    {
        _spriteBatch = spriteBatch;
    }

    public void ChangeScene(IScene newScene)
    {
        currentScene?.UnloadContent();
        currentScene = newScene;
        currentScene.LoadContent(_content, _spriteBatch);
    }

    public void Update(GameTime gameTime)
    {
        currentScene?.Update(gameTime);
    }

    public void Draw(GameTime gameTime)
    {
        currentScene?.Draw(gameTime);
    }
}
```

First of all I want to clear that what SpriteBatch means. It is a helper class used to draw visual things to the screen. That's all. ContentManager looks obvious I think.

I could make the ContentManager and SpriteBatch a global static reference from our Game1 class, but I want to keep things simple for now. You probably noticed that I initialize ContentManager in constructor, but I have a separate function for initializing SpriteBatch. This is because I'm going to initialize the SceneManager in our Game1's Initialize function, but at this time our SpriteBatch is not going to be initialized yet, which is going to be when LoadContent is called. So I'm going to initialize it in LoadContent.

Let's get back to our Game1 class. I created a variable for our scene manager. In the future, it is possible that we convert this to a global static reference to access it from everywhere easily. But right now we only need to use it in our Game1 class.

```cs
private SceneManager _sceneManager;
```

Then initialized it in our relevant function:

```cs
protected override void Initialize()
{
    _sceneManager = new SceneManager(Content);

    base.Initialize();
}
```

Notice that I write my own logic before the base call. Doing it afterwards caused problems. Probably there's some return statements or something like that.

After that, I also initialized my SpriteBatch to use in my scenes draw calls, in the LoadContent function:

```cs
protected override void LoadContent()
{
    _spriteBatch = new SpriteBatch(GraphicsDevice);

    _sceneManager.InitializeSpriteBatch(_spriteBatch);
}
```

Lastly, there goes our Update and Draw functions:

```cs
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

    _spriteBatch.Begin();

    _sceneManager.Draw(gameTime);

    _spriteBatch.End();

    base.Draw(gameTime);
}
```

In here, I call my SceneManager's Update function, and it does call the current scenes Update function. This is how I connect Game1's Update logic to my current scene.

Same goes with Draw calls. But before drawing, calling SpriteBatch's Begin function is necessary. After draw logic, calling the End function is necessary too. This is related to how MonoGame handles drawing. When you execute a draw call, you send information to GPU from CPU, then GPU processes this information and displays it. Sending these informations separately would be horrible for performance, so instead, MonoGame stores all of the information in RAM, then execute a draw call once with all of the information stored. This is why every Draw function is going to be between SpriteBatch's Begin and End calls.

Before testing, we need to create a scene. This is a simple TestScene class:

```cs
using Microsoft.Xna.Framework;
using Microsoft.Xna.Framework.Content;
using Microsoft.Xna.Framework.Graphics;

public class TestScene : IScene
{
    private ContentManager _content;
    private SpriteBatch _spriteBatch;

    public void LoadContent(ContentManager content, SpriteBatch spriteBatch)
    {
        _content = content;
        _spriteBatch = spriteBatch;
    }

    public void UnloadContent()
    {
        // Unload your scene content here
    }

    public void Update(GameTime gameTime)
    {
        // Update your scene logic here
    }

    public void Draw(GameTime gameTime)
    {
        // Draw your scene here
        _spriteBatch.DrawString(_content.Load<SpriteFont>("TextFont"), "Hello, MonoGame!",
        new Vector2(100, 100), Color.White);
    }
}
```

Notice that I load a SpriteFont called "TextFont" from ContentManager. [This](https://docs.monogame.net/articles/getting_to_know/howto/graphics/HowTo_Draw_Text.html) article explains how to draw a text. And I know this code is not that performance-friendly (heap allocation in parameters), but what I focus right now is demonstration.

Finally, for testing, I change my scene to TestScene from Game1's LoadContent function:

```cs
protected override void LoadContent()
{
    _spriteBatch = new SpriteBatch(GraphicsDevice);

    _sceneManager.InitializeSpriteBatch(_spriteBatch);
    _sceneManager.ChangeScene(new TestScene());
}
```

And here's the result:

![](/assets/scene-management.jpg)

In the next part, we're going to look at ingame entities known as "game objects".