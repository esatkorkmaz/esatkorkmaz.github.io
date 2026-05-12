---
title: 'Game Dev Fundamentals Series #1: Introduction'
author: Esat Korkmaz
date: 2026-05-12
category: gamedev-fundamentals
layout: post
---

Welcome to Game Dev Fundamentals Series. My aim on this series is adopting engine-agnostic game development with covering topics such as gameplay mechanics and systems, AI and user interface.

Although modern game engines are very popular and efficient, I think it's still important to lean towards some old school game development. This doesn't mean writing things entirely from scratch like rendering and physics, but being able to adapt industry standards independent from specific implementations of game engines. This is important, because otherwise you're in risk being vulnerable to external factors affecting your development cycle negatively. A well known example is Unity's 2023 Runtime Fee controversy.

In this series, I'm not going to cover engine or engine-related programming. Instead, as I mentioned above, my focus is going to be on things related to "higher-level", more "game-oriented" stuff. In this case, I'm going to use the MonoGame framework. This is a personal blog and does not serve an educational purpose but, if you still wanna dive into it, I want to say that prior game development experience is not required to benefit from this series, but it would be a huge plus and I'm also going to assume you installed MonoGame, created a new project and have somehow a programming experience. If you know NOTHING about game development and programming, this isn't for you.

#### What is and why MonoGame?

MonoGame is a game development framework and a fork of Microsoft's discontinued XNA framework. It is written in C# and it's a battle-tested framework. Many games such as Terraria, Stardew Valley, Celeste, Carrion, Fez, Bastion and Barotrauma built with MonoGame or XNA.

MonoGame handles the graphics, input and audio management, and also provides a barebones architecture (update and draw calls etc.) for you. The rest is all yours. This means it's a very minimal tool compared to a game engine like Unity, Unreal or Godot, but also more convenient compared to options like SDL which you have to do things by yourself that MonoGame handles for you too. This puts MonoGame on an ideal position for people who don't want to spend time on things that close to the metal, but also want to have full control on the architecture compared to the more advanced game engines. Like me, haha.

An other option I considered was raylib, which is excellent for educational purposes. But it's way too simple compared to a framework like MonoGame and it's more suitable for total beginners or maybe hobbyists. MonoGame is a more structured option than raylib, but that doesn't mean it's worse for learning. It's a proven framework and a solid choice for my purposes. Also, C# is a suitable programming language in my case. Without dealing with things like memory management, it allows us to focus on systems with a clean architecture. We're not going to have a business with performance-critical things that every milliseconds is relatively important like handling thousands of game objects. For my purposes, it is more than enough. But I still use raylib with C/C++ to practice topics such as data structures and algorithms.

In the next part, we're going to look at scene management in MonoGame.