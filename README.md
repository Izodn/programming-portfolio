# Brandon Burton - Programming Portfolio (samples)

## Introduction

Hello there!
My name is Brandon Burton, and this is a Markdown file containing what I consider to be a portfolio.  
Below, you'll notice a number of files which represent real-world coding from me.  
The examples given are part of a work-in-progress, so this really does represent the kind of coding I do.

Why did I include references to PSP instead of OpenGL, Vulkan, Direct X, or any other modern technology?  
Simple! PSP development documentation is far more rare and therefore more difficult to learn.  
I included PSP references to demonstrate that I'm capable of learning slightly more archaic systems.

## Game Engine

<details>
	<summary>Source Code</summary>

<details>
	<summary>Makefile</summary>

```Makefile
TARGET_EXEC ?= engine
CPUS ?= $(shell nproc)
MAKEFLAGS += -s --jobs=$(CPUS)

BUILD_DIR ?= ./bin
SRC_DIRS ?= ./src

SRCS := $(shell find $(SRC_DIRS) -name *.cpp)
OBJS := $(SRCS:%=$(BUILD_DIR)/%.o)
DEPS := $(OBJS:.o=.d)

INC_DIRS := ../libraries
INC_FLAGS := $(addprefix -I,$(INC_DIRS))

LDFLAGS := -static -L../libraries/asio -lws2_32 -L../libraries/GL -L../libraries/GLFW -L../libraries/SOIL -lglew32 -lglfw3 -lSOIL -lopengl32 -lgdi32 -lpthread

CPPFLAGS ?= -std=c++14 $(INC_FLAGS) -D_WIN32_WINNT=0x0601

all: debug

profile: EXTRACPPFLAGS += -DPROFILER
profile: debug

debug: EXTRACPPFLAGS += -g -O0
debug: _all

release: EXTRACPPFLAGS += -g0 -O3
release: EXTRALDFLAGS += -flto -Wl,-s -Wl,--strip-all
release: _all

_all: $(OBJS)
	$(AR) -rcs $(BUILD_DIR)/lib$(TARGET_EXEC).a $(OBJS)

$(BUILD_DIR)/%.cpp.o: %.cpp
	$(MKDIR_P) $(dir $@)
	$(CXX) $(CPPFLAGS) $(EXTRACPPFLAGS) -c $< -o $@

...
```
</details>

<details>
	<summary>src/Game.hpp</summary>

```c++
#pragma once

#include <cstdint>

#include "GameMode.hpp"
#include "graphics/Renderer.hpp"

class CameraComponent;
class EventHandler;
class GameLevel;
class HeadlessApplication;
class NetworkHandler;
class NetworkServer;
class OpenGLApplication;
class Timer;
class Window;

class Game
{
	protected:
		#ifndef __PSP__
			NetworkHandler* m_NetworkHandler = nullptr;
		#endif
		EventHandler* m_EventHandler = nullptr;
		Renderer* m_Renderer = nullptr;
		Window* m_Window = nullptr;
		GameLevel* m_CurrentLevel = nullptr;
		CameraComponent* m_Camera = nullptr;
		bool m_Running;
		uint64_t m_CurrentFrame;
		uint64_t m_UpdateTickRate;
		uint64_t m_LastUpdateTick;
		Timer* m_Timer;
		GameMode m_GameMode = GameMode::CLIENT;

	public:
		Game(GameMode);
		void Run();
		void Stop();
		void ChangeToLevel(GameLevel*);
		void SetCameraComponent(CameraComponent*);
		CameraComponent* GetCameraComponent();
		void Cleanup();
		uint64_t GetTickRate();
		void UpdateTickRate(uint64_t);
		Window* GetWindow();
		#ifndef __PSP__
			NetworkHandler* GetNetworkHandler();
			void SetNetworkHandler(NetworkHandler*);
		#endif
		void SetGameMode(GameMode);
		GameMode GetGameMode();
		bool IsRunning();

	public:
		template<typename T>
		void RegisterShader()
		{
			m_Renderer->RegisterShader<T>();
		};
};
```
</details>

<details>
	<summary>src/Game.hpp</summary>

```c++
#include "core/Input.hpp"
#include "core/Logger.hpp"
#include "core/Timer.hpp"
#include "EventHandler.hpp"
#include "GameLevel.hpp"
#include "graphics/Renderer.hpp"
#include "graphics/Window.hpp"
#include "headless/HeadlessApplication.hpp"
#include "NetworkHandler.hpp"
#include "profiler/Profiler.hpp"

#ifdef __PSP__
	#include <pspkernel.h>
	#include "psp/PSPApplication.hpp"
#else
	#include "networking/NetworkClient.hpp"
	#include "networking/NetworkServer.hpp"
	#include "opengl/OpenGLApplication.hpp"
#endif

#include "Game.hpp"

Game::Game(GameMode mode) :
	m_Running(false),
	m_CurrentFrame(0),
	m_UpdateTickRate(120),
	m_LastUpdateTick(0),
	m_Timer(Timer::Instance()),
	m_GameMode(mode)
{
	if (mode == GameMode::SERVER) {
		HeadlessApplication app;
		app.Setup(this, &m_EventHandler, &m_Renderer, &m_Window);
	} else if (mode == GameMode::CLIENT) {
		#ifdef __PSP__
			PSPApplication app;
		#else
			OpenGLApplication app;
		#endif
		app.Setup(this, &m_EventHandler, &m_Renderer, &m_Window);
	}
}

void Game::Run()
{
	m_Running = true;
	uint64_t lastFpsOut = 0;
	uint64_t lastFpsFrame = 0;

	#ifndef __PSP__
		if (m_NetworkHandler != nullptr) {
			if (m_GameMode == GameMode::SERVER) {
				Logger() << "Starting server" << "\n";
				m_NetworkHandler->GetNetworkServer()->StartServer(this);
			} else if (m_NetworkHandler != nullptr) {
				Logger() << "Starting server listener (client)" << "\n";
				m_NetworkHandler->GetNetworkClient()->StartListener(this);
			}
		}
	#endif

	while (m_Running) {
		Profiler::Instance()->Reset();
		Profiler::Instance()->Start("Game loop");
		{
			m_CurrentFrame++;

			// Poll events
			Profiler::Instance()->Start("Poll events");
			{
				m_EventHandler->PollEvents();
			}
			Profiler::Instance()->Stop("Poll events");

			// Update timer
			uint64_t tick = m_Timer->GetTick();
			uint64_t deltaTick = tick - m_LastUpdateTick;
			if (deltaTick >= 1000 / m_UpdateTickRate) {
				Profiler::Instance()->Start("Update objects");
				{
					m_CurrentLevel->UpdateObjects(deltaTick / 1000.0f);
				}
				Profiler::Instance()->Stop("Update objects");

				m_LastUpdateTick = tick;

				// Sync input handling for correct pressed/released triggers
				Input::Sync();
				Input::ClearReleased();
				Input::ClearCursorDelta();

				if (m_Window->IsCursorLocked()) {
					m_EventHandler->ResetCursor();
				}
			}

			// If we have a camera ready to render
			if (m_GameMode != GameMode::SERVER) {
				if (m_Camera != nullptr) {
					// Prepare the window for drawing
					Profiler::Instance()->Start("Prepare renderer");
					{
						m_Renderer->Prepare();
					}
					Profiler::Instance()->Stop("Prepare renderer");

					// Render objects
					Profiler::Instance()->Start("Render level objects");
					{
						m_CurrentLevel->RenderUpdateObjects();
					}
					Profiler::Instance()->Stop("Render level objects");

					// Flush the graphics to the window
					Profiler::Instance()->Start("Flush renderer");
					{
						m_Renderer->Flush();
					}
					Profiler::Instance()->Stop("Flush renderer");
				} else {
					Logger() << "Cannot render; No camera.\n";
				}
			} else {
				// This will sleep the thread for at least 1ms
				// A poor attempt at reducing CPU usage for the server
				// At the expense of
				//   - Unpredictable wait times per loop
				//   - Relying on the OS Scheduler
				//   - No ability to run beyond the min sleep time determined by OS scheduler
				//     - In testing, around 15.5ms
				m_Timer->Delay(1);
			}

			// Output FPS
			if ((tick - lastFpsOut) >= 1000) {
				double fps = (m_CurrentFrame - lastFpsFrame);
				fps /= (tick - lastFpsOut) / 1000;

				Logger() << "FPS: " << fps << "\n";
				lastFpsOut = tick;
				lastFpsFrame = m_CurrentFrame;
			}
		}
		Profiler::Instance()->Stop("Game loop");
		Profiler::Instance()->Output();
	}

	Logger() << "Initiating cleanup" << "\n";
	Cleanup();
	Logger() << "Game::Run(): Cleanup complete" << "\n";
}

void Game::Stop()
{
	Logger() << "Game::Stop() called" << "\n";
	m_Running = false;
}

void Game::ChangeToLevel(GameLevel* level)
{
	Logger() << "Changing to level: " << (void*)level << "\n";

	// If we're on a level currently, tell the old level to cleanup
	if (m_CurrentLevel != nullptr) {
		m_CurrentLevel->Cleanup();
	}

	if (level != nullptr) {
		// Set the new level
		m_CurrentLevel = level;
		m_CurrentLevel->SetGame(this);
		m_CurrentLevel->Init();
		m_CurrentLevel->StartObjects();
	}

	Logger() << "Change to level successful" << "\n";
}

void Game::SetCameraComponent(CameraComponent* camera)
{
	m_Camera = camera;
}

CameraComponent* Game::GetCameraComponent()
{
	return m_Camera;
}

void Game::Cleanup()
{
	// Logger() << "Game::Cleanup()" << "\n";
	if (m_CurrentLevel != nullptr) {
		// Logger() << "Cleaning up level" << "\n";
		// Trigger cleanup
		m_CurrentLevel->Cleanup();

		// Actually free the memory
		// Logger() << "Deleting level: " << (void*)m_CurrentLevel << "\n";
		delete m_CurrentLevel;
		// Logger() << "Level successfully deleted" << "\n";
	} else {
		Logger() << "Level was not set, continuing" << "\n";
	}
	m_Renderer->Cleanup();
	Profiler::Instance()->Reset();
	#ifdef __PSP__
		sceKernelExitGame();
	#endif
	// Logger() << "Game::Cleanup(): Cleanup complete" << "\n";
}

void Game::UpdateTickRate(uint64_t tickRate)
{
	// Game tick rate in MS
	m_UpdateTickRate = tickRate;
	if (m_UpdateTickRate > 1000 || m_UpdateTickRate == 0) {
		m_UpdateTickRate = 1000;
	}
}

uint64_t Game::GetTickRate()
{
	return m_UpdateTickRate;
}

Window* Game::GetWindow()
{
	return m_Window;
}

#ifndef __PSP__
	NetworkHandler* Game::GetNetworkHandler()
	{
		return m_NetworkHandler;
	}

	void Game::SetNetworkHandler(NetworkHandler* networkHandler)
	{
		m_NetworkHandler = networkHandler;
	}
#endif

void Game::SetGameMode(GameMode mode)
{
	m_GameMode = mode;
}

GameMode Game::GetGameMode()
{
	return m_GameMode;
}

bool Game::IsRunning()
{
	return m_Running;
}
```
</details>

<details>
	<summary>src/GameObject.hpp</summary>

```c++
#pragma once

#include <string>

#include "core/Collection.hpp"

class Component;
class GameLevel;
class RenderComponent;
class Transform;

class GameObject
{
	private:
		std::string m_Name;
		Collection<Component> m_Components;
		Collection<RenderComponent> m_RenderComponents;
		Transform* m_Transform;
		GameLevel* m_GameLevel;

	public:
		GameObject();
		void SetName(std::string);
		std::string GetName();
		Collection<Component>* Components();
		Collection<RenderComponent>* RenderComponents();
		Transform* GetTransform();
		void SetLevel(GameLevel*);
		GameLevel* GetLevel();
		void Start();
		void Update(double);
		void RenderUpdate();
		void Cleanup();
};
```
</details>

<details>
	<summary>src/GameObject.cpp</summary>

```c++
#include "component/Component.hpp"
#include "component/render/RenderComponent.hpp"
#include "component/Transform.hpp"
#include "core/Logger.hpp"
#include "Game.hpp"
#include "GameLevel.hpp"
#include "graphics/Window.hpp"

#include "GameObject.hpp"

GameObject::GameObject()
{
	m_Components.Add<Transform>();
	m_Transform = m_Components.Get<Transform>();
}

void GameObject::SetLevel(GameLevel* gameLevel)
{
	m_GameLevel = gameLevel;
}

GameLevel* GameObject::GetLevel()
{
	return m_GameLevel;
}

void GameObject::Start()
{
	// Generic components
	for (Component* component : m_Components.GetAll()) {
		component->SetGameObject(this);
		component->Start();
	}

	// If running as server, ignore render updates
	if (GetLevel()->GetGame()->GetGameMode() != GameMode::SERVER) {
		// Render components
		for (RenderComponent* component : m_RenderComponents.GetAll()) {
			component->SetGameObject(this);
			component->SetRenderer(
				m_GameLevel->GetGame()->GetWindow()->GetRenderer()
			);
			component->Start();
		}
	}
}

void GameObject::Update(double deltaTime)
{
	// Generic components
	for (Component* component : m_Components.GetAll()) {
		if (component->IsEnabled()) {
			component->Update(deltaTime);
		}
	}

	// If running as server, ignore render updates
	if (GetLevel()->GetGame()->GetGameMode() != GameMode::SERVER) {
		// Render components
		for (RenderComponent* component : m_RenderComponents.GetAll()) {
			if (component->IsEnabled()) {
				component->Update(deltaTime);
			}
		}
	}

}

void GameObject::RenderUpdate()
{
	for (RenderComponent* component : m_RenderComponents.GetAll()) {
		component->Render();
	}
}

void GameObject::SetName(std::string name)
{
	m_Name = name;
}

std::string GameObject::GetName()
{
	return m_Name;
}

Collection<Component>* GameObject::Components()
{
	return &m_Components;
}

Collection<RenderComponent>* GameObject::RenderComponents()
{
	return &m_RenderComponents;
}

Transform* GameObject::GetTransform()
{
	return m_Transform;
}

void GameObject::Cleanup()
{
	// Generic components
	for (Component* component : m_Components.GetAll()) {
		if (component != NULL) {
			// Logger() << "Deleting component: " << (void*)component << "\n";
			delete component;
			// Logger() << "Deleted component" << "\n";
		}
	}

	// If running as server, ignore render updates
	if (GetLevel()->GetGame()->GetGameMode() != GameMode::SERVER) {
		// Render components
		for (RenderComponent* component : m_RenderComponents.GetAll()) {
			if (component != NULL) {
				// Logger() << "Deleting render component: " << (void*)component << "\n";
				delete component;
				// Logger() << "Deleted component" << "\n";
			}
		}
	}
}

```
</details>

<details>
	<summary>src/GameLevel.hpp</summary>

```c++
#pragma once

#include <string>
#include <vector>

class Game;
class GameObject;

class GameLevel
{
	protected:
		std::vector<GameObject*>* m_GameObjects = new std::vector<GameObject*>();
		std::vector<GameObject*>* m_GameServerObjects = new std::vector<GameObject*>();
		Game* m_Game;

	public:
		void StartObjects();
		void UpdateObjects(double);
		void RenderUpdateObjects();
		void Cleanup();
		void SetGame(Game*);
		Game* GetGame();
		std::vector<GameObject*>* GetGameObjects();
		GameObject* GetObjectByName(std::string);

	public:
		virtual void Init() = 0;
};
```
</details>

<details>
	<summary>src/GameLevel.cpp</summary>

```c++
#include "core/Logger.hpp"
#include "Game.hpp"
#include "GameMode.hpp"
#include "GameObject.hpp"

#include "GameLevel.hpp"

void GameLevel::SetGame(Game* game)
{
	m_Game = game;
}

Game* GameLevel::GetGame()
{
	return m_Game;
}

std::vector<GameObject*>* GameLevel::GetGameObjects()
{
	return m_GameObjects;
}

void GameLevel::StartObjects()
{
	Logger() << "GameLevel::StartObjects" << "\n";
	for (GameObject* object : *m_GameObjects) {
		if (object != NULL) {
			object->SetLevel(this);
			object->Start();
		} else {
			Logger() << "Tried to start NULL GameObject" << "\n";
		}
	}

	if (m_Game->GetGameMode() == GameMode::SERVER) {
		for (GameObject* object : *m_GameServerObjects) {
			if (object != NULL) {
				object->SetLevel(this);
				object->Start();
			} else {
				Logger() << "Tried to start NULL GameObject" << "\n";
			}
		}
	}
}

void GameLevel::UpdateObjects(double deltaTime)
{
	for (GameObject* object : *m_GameObjects) {
		if (object != NULL) {
			object->Update(deltaTime);
		} else {
			Logger() << "Tried to update NULL GameObject" << "\n";
		}
	}

	if (m_Game->GetGameMode() == GameMode::SERVER) {
		for (GameObject* object : *m_GameServerObjects) {
			if (object != NULL) {
				object->Update(deltaTime);
			} else {
				Logger() << "Tried to update NULL GameObject" << "\n";
			}
		}
	}
}

void GameLevel::RenderUpdateObjects()
{
	if (m_Game->GetGameMode() == GameMode::SERVER) {
		return;
	}

	for (GameObject* object : *m_GameObjects) {
		if (object != NULL) {
			object->RenderUpdate();
		} else {
			Logger() << "Tried to render update NULL GameObject" << "\n";
		}
	}
}

void GameLevel::Cleanup()
{
	// Logger() << "GameLevel::Cleanup()" << "\n";
	for (GameObject* object : *m_GameObjects) {
		if (object != NULL) {
			// Trigger object cleanup
			object->Cleanup();

			// Actually free memory
			// Logger() << "Deleting object: " << (void*)object << "\n";
			delete object;
			// Logger() << "Object successfully deleted" << "\n";
		} else {
			Logger() << "Tried to remove NULL GameObject" << "\n";
		}
	}

	for (GameObject* object : *m_GameServerObjects) {
		if (object != NULL) {
			// Trigger object cleanup
			object->Cleanup();

			// Actually free memory
			// Logger() << "Deleting server object: " << (void*)object << "\n";
			delete object;
			// Logger() << "Object successfully deleted" << "\n";
		} else {
			Logger() << "Tried to remove NULL GameObject" << "\n";
		}
	}
}

GameObject* GameLevel::GetObjectByName(std::string name)
{
	// I'm sure we'll want to handle server objects somehow...
	for (GameObject* object : *m_GameObjects) {
		if (object != NULL) {
			if (object->GetName() == name) {
				return object;
			}
		}
	}
	return nullptr;
}
```
</details>

<details>
	<summary>src/maths/Vector3.hpp</summary>

```c++
#pragma once

#include <string>

class Vector3
{
	// Members
	protected:
		double m_X;
		double m_Y;
		double m_Z;

	// Constructors
	public:
		Vector3(double, double, double);
		Vector3();

	// Getters/Setters
	public:
		Vector3 SetX(double);
		Vector3 SetY(double);
		Vector3 SetZ(double);
		double GetX() const;
		double GetY() const;
		double GetZ() const;

	// Default Vectors
	public:
		static Vector3 Forward();
		static Vector3 Up();
		static Vector3 Right();

	// Static methods
	public:
		static double Length(const Vector3&);
		static Vector3 Normalized(const Vector3&);
		static Vector3 Normalize(Vector3);
		static double Distance(const Vector3&, const Vector3&);
		static double Angle(const Vector3&, const Vector3&);
		static double Dot(const Vector3&, const Vector3&);
		static Vector3 Cross(const Vector3&, const Vector3&);

	// Methods
	public:
		double Length() const;
		Vector3 Normalized() const;
		Vector3 Normalize();
		double Distance(const Vector3&) const;
		double Angle(const Vector3&) const;
		double Dot(const Vector3&) const;
		Vector3 Cross(const Vector3&) const;

	// Operators
	public:
		Vector3 operator=(const Vector3&);

		Vector3 operator+(const Vector3&) const;
		Vector3 operator+(double) const;
		Vector3 operator+=(const Vector3&);
		Vector3 operator+=(double);

		Vector3 operator-(const Vector3&) const;
		Vector3 operator-(double) const;
		Vector3 operator-=(const Vector3&);
		Vector3 operator-=(double);

		Vector3 operator*(const Vector3&) const;
		Vector3 operator*(double) const;
		Vector3 operator*=(const Vector3&);
		Vector3 operator*=(double);

		Vector3 operator/(const Vector3&) const;
		Vector3 operator/(double) const;
		Vector3 operator/=(const Vector3&);
		Vector3 operator/=(double);

		bool operator==(const Vector3&) const;
		bool operator!=(const Vector3&) const;

		operator std::string() const;
};
```
</details>

<details>
	<summary>src/maths/Vector3.cpp</summary>

```c++
#define _USE_MATH_DEFINES
#include <cmath>
#include <sstream>

#include "Vector3.hpp"

// Constructors
Vector3::Vector3(double x, double y, double z)
{
	SetX(x);
	SetY(y);
	SetZ(z);
}
Vector3::Vector3()
{
	Vector3(0.0f, 0.0f, 0.0f);
}

// Getters/Setters
Vector3 Vector3::SetX(double x)
{
	m_X = x;
	return *this;
}
Vector3 Vector3::SetY(double y)
{
	m_Y = y;
	return *this;
}
Vector3 Vector3::SetZ(double z)
{
	m_Z = z;
	return *this;
}
double Vector3::GetX() const
{
	return m_X;
}
double Vector3::GetY() const
{
	return m_Y;
}
double Vector3::GetZ() const
{
	return m_Z;
}

// Default Vectors
Vector3 Vector3::Forward()
{
	return Vector3(0.0f, 0.0f, 1.0f);
}
Vector3 Vector3::Up()
{
	return Vector3(0.0f, 1.0f, 0.0f);
}
Vector3 Vector3::Right()
{
	return Vector3(1.0f, 0.0f, 0.0f);
}

// Static methods
double Vector3::Length(const Vector3& vec)
{
	return sqrt(
		pow(vec.GetX(), 2.0f)
		+ pow(vec.GetY(), 2.0f)
		+ pow(vec.GetZ(), 2.0f)
	);
}
Vector3 Vector3::Normalized(const Vector3& vec)
{
	return vec / vec.Length();
}
Vector3 Vector3::Normalize(Vector3 vec)
{
	return vec = Normalized(vec);
}
double Vector3::Distance(const Vector3& vec1, const Vector3& vec2)
{
	return (vec1 - vec2).Length();
}
double Vector3::Angle(const Vector3& vec1, const Vector3& vec2)
{
	double denominator = (vec1.Length() * vec2.Length());

	// if devide by zero, then must be a 90-degree angle
	if (denominator == 0.0f) {
		return 0.0f;
	}

	return acos((Vector3::Dot(vec1, vec2)) / denominator);
}
double Vector3::Dot(const Vector3& vec1, const Vector3& vec2)
{
	return (
		(vec1.GetX() * vec2.GetX()) +
		(vec1.GetY() * vec2.GetY()) +
		(vec1.GetZ() * vec2.GetZ())
	);
}
Vector3 Vector3::Cross(const Vector3& vec1, const Vector3& vec2)
{
	return Vector3(
		vec1.GetY() * vec2.GetZ() - vec1.GetZ() * vec2.GetY(),
		vec1.GetZ() * vec2.GetX() - vec1.GetX() * vec2.GetZ(),
		vec1.GetX() * vec2.GetY() - vec1.GetY() * vec2.GetX()
	);
}

// Methods
double Vector3::Length() const
{
	return Length(*this);
}
Vector3 Vector3::Normalize()
{
	return Normalize(*this);
}
Vector3 Vector3::Normalized() const
{
	return Normalized(*this);
}
double Vector3::Distance(const Vector3& target) const
{
	return Distance(*this, target);
}
double Vector3::Angle(const Vector3& target) const
{
	return Angle(*this, target);
}
double Vector3::Dot(const Vector3& target) const
{
	return Dot(*this, target);
}
Vector3 Vector3::Cross(const Vector3& target) const
{
	return Cross(*this, target);
}

// Operators
Vector3 Vector3::operator=(const Vector3& rh)
{
	m_X = rh.GetX();
	m_Y = rh.GetY();
	m_Z = rh.GetZ();
	return *this;
}

Vector3 Vector3::operator+(const Vector3& rh) const
{
	return Vector3(
		m_X + rh.GetX(),
		m_Y + rh.GetY(),
		m_Z + rh.GetZ()
	);
}
Vector3 Vector3::operator+(double rh) const
{
	return *this + Vector3(rh, rh, rh);
}
Vector3 Vector3::operator+=(const Vector3& rh)
{
	return *this = *this + rh;
}
Vector3 Vector3::operator+=(double rh)
{
	return *this = *this + rh;
}

Vector3 Vector3::operator-(const Vector3& rh) const
{
	return Vector3(
		m_X - rh.GetX(),
		m_Y - rh.GetY(),
		m_Z - rh.GetZ()
	);
}
Vector3 Vector3::operator-(double rh) const
{
	return *this - Vector3(rh, rh, rh);
}
Vector3 Vector3::operator-=(const Vector3& rh)
{
	return *this = *this - rh;
}
Vector3 Vector3::operator-=(double rh)
{
	return *this = *this - rh;
}

Vector3 Vector3::operator*(const Vector3& rh) const
{
	return Vector3(
		m_X * rh.GetX(),
		m_Y * rh.GetY(),
		m_Z * rh.GetZ()
	);
}
Vector3 Vector3::operator*(double rh) const
{
	return *this * Vector3(rh, rh, rh);
}
Vector3 Vector3::operator*=(const Vector3& rh)
{
	return *this = *this * rh;
}
Vector3 Vector3::operator*=(double rh)
{
	return *this = *this * rh;
}

Vector3 Vector3::operator/(const Vector3& rh) const
{
	return Vector3(
		m_X / rh.GetX(),
		m_Y / rh.GetY(),
		m_Z / rh.GetZ()
	);
}
Vector3 Vector3::operator/(double rh) const
{
	return *this / Vector3(rh, rh, rh);
}
Vector3 Vector3::operator/=(const Vector3& rh)
{
	return *this = *this / rh;
}
Vector3 Vector3::operator/=(double rh)
{
	return *this = *this / rh;
}

bool Vector3::operator==(const Vector3& rh) const
{
	return (
		m_X == rh.GetX()
		&& m_Y == rh.GetY()
		&& m_Z == rh.GetZ()
	);
}
bool Vector3::operator!=(const Vector3& rh) const
{
	return !(*this == rh);
}

Vector3::operator std::string() const
{
	std::ostringstream strX;
	std::ostringstream strY;
	std::ostringstream strZ;
	strX << m_X;
	strY << m_Y;
	strZ << m_Z;

	std::string retStr = "Vector3(" +
		strX.str() + "f, " +
		strY.str() + "f, " +
		strZ.str() + "f)";
	return retStr;
}
```
</details>

<details>
	<summary>src/maths/Quaternion.hpp</summary>

```c++
#pragma once

#include <string>

class Vector3;

class Quaternion
{
	// Members
	protected:
		double m_X;
		double m_Y;
		double m_Z;
		double m_W;

	// Constructors
	public:
		Quaternion();
		Quaternion(double);
		Quaternion(const Vector3&, double);
		Quaternion(const Vector3&);
		Quaternion(double, double, double, double);

	// Getters/Setters
	public:
		Quaternion SetX(double);
		Quaternion SetY(double);
		Quaternion SetZ(double);
		Quaternion SetW(double);
		double GetX() const;
		double GetY() const;
		double GetZ() const;
		double GetW() const;

	// Static methods
	public:
		static Quaternion Identity();
		static double Norm(const Quaternion&);
		static double Length(const Quaternion&);
		static Quaternion Normalized(const Quaternion&);
		static Quaternion Normalize(Quaternion);
		static Quaternion FromAxisAngle(const Vector3&, double);
		static Quaternion FromEulerAngles(const Vector3&);
		static Vector3 ToEulerAngles(const Quaternion&);
		static Vector3 GetAxis(const Quaternion&);
		static Quaternion Rotation(const Vector3&, const Vector3&);
		static Quaternion Rotation(double, const Vector3&);
		static Vector3 Rotate(const Quaternion&, const Vector3&);
		static Quaternion Rotate(const Quaternion&, const Vector3&, double);
		static double Dot(const Quaternion&, const Quaternion&);
		static Quaternion LookAt(const Vector3&, const Vector3&);
		static double Angle(const Quaternion&);

	// Methods
	public:
		double Norm() const;
		double Length() const;
		Quaternion Normalized() const;
		Quaternion Normalize();
		Vector3 ToEulerAngles() const;
		Vector3 GetAxis() const;
		Vector3 Rotate(const Vector3&) const;
		Quaternion Rotate(const Vector3&, double);
		double Dot(const Quaternion&) const;
		double Angle() const;

	// Operators
	public:
		Quaternion operator=(const Quaternion&);

		Quaternion operator+(const Quaternion&) const;
		Quaternion operator+=(const Quaternion&);

		Quaternion operator-(const Quaternion&) const;
		Quaternion operator-=(const Quaternion&);

		Quaternion operator*(const Quaternion&) const;
		Quaternion operator*=(const Quaternion&);

		Quaternion operator*(double) const;
		Quaternion operator*=(double);

		Quaternion operator/(double) const;
		Quaternion operator/=(double);

		Vector3 operator*(const Vector3&) const;

		bool operator==(const Quaternion&) const;
		bool operator!=(const Quaternion&) const;

		operator std::string() const;
};
```
</details>

<details>
	<summary>src/maths/Quaternion.cpp</summary>

```c++
#define _USE_MATH_DEFINES
#include <cmath>
#include <sstream>

#include "Vector3.hpp"
#include "Quaternion.hpp"

// Constructors
Quaternion::Quaternion()
{
	Quaternion(0.0f, 0.0f, 0.0f, 1.0f);
}
Quaternion::Quaternion(double scalar)
{
	Quaternion(scalar, scalar, scalar, scalar);
}
Quaternion::Quaternion(const Vector3& vec, double w)
{
	Quaternion(vec.GetX(), vec.GetY(), vec.GetZ(), w);
}
Quaternion::Quaternion(const Vector3& vec)
{
	Quaternion quat = Quaternion::FromEulerAngles(vec);
	Quaternion(quat.GetX(), quat.GetY(), quat.GetZ(), quat.GetW());
}
Quaternion::Quaternion(double x, double y, double z, double w)
{
	m_X = x;
	m_Y = y;
	m_Z = z;
	m_W = w;
}

// Getters/Setters
Quaternion Quaternion::SetX(double x)
{
	m_X = x;
	return *this;
}
Quaternion Quaternion::SetY(double y)
{
	m_Y = y;
	return *this;
}
Quaternion Quaternion::SetZ(double z)
{
	m_Z = z;
	return *this;
}
Quaternion Quaternion::SetW(double w)
{
	m_W = w;
	return *this;
}
double Quaternion::GetX() const
{
	return m_X;
}
double Quaternion::GetY() const
{
	return m_Y;
}
double Quaternion::GetZ() const
{
	return m_Z;
}
double Quaternion::GetW() const
{
	return m_W;
}

// Static methods
Quaternion Quaternion::Identity()
{
	return Quaternion(
		0.0f,
		0.0f,
		0.0f,
		1.0f
	);
}
double Quaternion::Norm(const Quaternion& quat)
{
	return (
		(quat.GetX() * quat.GetX())
		+ (quat.GetY() * quat.GetY())
		+ (quat.GetZ() * quat.GetZ())
		+ (quat.GetW() * quat.GetW())
	);
}
double Quaternion::Length(const Quaternion& quat)
{
	return sqrt(Norm(quat));
}
Quaternion Quaternion::Normalized(const Quaternion& quat)
{
	return quat * (1.0f / Norm(quat));
}
Quaternion Quaternion::Normalize(Quaternion quat)
{
	return quat = Normalized(quat);
}
Quaternion Quaternion::FromAxisAngle(const Vector3& axis, double angle)
{
	double halfAngle = angle / 2.0f;
	double s = sin(halfAngle);
	return Quaternion(
		axis.GetX() * s,
		axis.GetY() * s,
		axis.GetZ() * s,
		cos(halfAngle)
	);
}
Quaternion Quaternion::FromEulerAngles(const Vector3& vec)
{
	double vX, vY, vZ, t0, t1, t2, t3, t4, t5;
	vX = vec.GetX();
	vY = vec.GetY();
	vZ = vec.GetZ();

	t0 = cos(vX * 0.5f);
	t1 = sin(vX * 0.5f);
	t2 = cos(vY * 0.5f);
	t3 = sin(vY * 0.5f);
	t4 = cos(vZ * 0.5f);
	t5 = sin(vZ * 0.5f);

	return Quaternion(
		/*X*/ t1 * t2 * t4 + t0 * t3 * t5,
		/*Y*/ t0 * t3 * t4 - t1 * t2 * t5,
		/*Z*/ t0 * t2 * t5 + t1 * t3 * t4,
		/*W*/ t0 * t2 * t4 - t1 * t3 * t5
	);
}
Vector3 Quaternion::ToEulerAngles(const Quaternion& quat)
{
	double qX, qY, qZ, qW, t0, t1, t2, t3, t4;
	qX = quat.GetX();
	qY = quat.GetY();
	qZ = quat.GetZ();
	qW = quat.GetW();

	t0 = 2.0f         * ((qW * qX) - (qY * qZ)) ; // Pitch
	t1 = 1.0f - (2.0f * ((qX * qX) + (qY * qY))); // Pitch
	t2 = 2.0f         * ((qW * qY) + (qZ * qX)) ; // Yaw
	t3 = 2.0f         * ((qW * qZ) - (qX * qY)) ; // Roll
	t4 = 1.0f - (2.0f * ((qY * qY) + (qZ * qZ))); // Roll

	// Gimble lock check
	t2 = t2 >  1.0f ?  1.0f : t2;
	t2 = t2 < -1.0f ? -1.0f : t2;

	return Vector3(
		/*Pitch*/ atan2(t0, t1),
		/*Yaw*/ asin(t2),
		/*Roll*/ atan2(t3, t4)
	);
}
Vector3 Quaternion::GetAxis(const Quaternion& quat)
{
	double x1 = 1.0f - quat.GetW() * quat.GetW();
	double x2 = x1 * x1;

	// Devide by 0 check
	if (x2 == 0.0f) {
		return Vector3::Forward();
	}

	return Vector3(quat.GetX(), quat.GetY(), quat.GetZ()) / x2;
}
Quaternion Quaternion::Rotation(const Vector3& uVec1, const Vector3& uVec2)
{
	double cosHalfAngleX2 = sqrt(
		(2.0f * (1.0f + uVec1.Dot(uVec2)))
	);
	double recipCosHalfAngleX2 = (1.0f / cosHalfAngleX2);
	return Quaternion(
		(uVec1.Cross(uVec2) * recipCosHalfAngleX2),
		(cosHalfAngleX2 * 0.5f)
	);
}
Quaternion Quaternion::Rotation(double radians, const Vector3& uVec)
{
	double angle = radians * 0.5f;
	return Quaternion((uVec * sin(angle)), cos(angle));
}
Vector3 Quaternion::Rotate(const Quaternion& quat, const Vector3& vec)
{
	double qX = quat.GetX();
	double qY = quat.GetY();
	double qZ = quat.GetZ();
	double qW = quat.GetW();
	double vX = vec.GetX();
	double vY = vec.GetY();
	double vZ = vec.GetZ();

	double x = (((qW * vX) + (qY * vZ)) - (qZ * vY));
	double y = (((qW * vY) + (qZ * vX)) - (qX * vZ));
	double z = (((qW * vZ) + (qX * vY)) - (qY * vX));
	double w = (((qX * vX) + (qY * vY)) + (qZ * vZ));

	return Vector3(
		((((w * qX) + (x * qW)) - (y * qZ))+ (z * qY)),
		((((w * qY) + (y * qW)) - (z * qX))+ (x * qZ)),
		((((w * qZ) + (z * qW)) - (x * qY))+ (y * qX))
	);
}
Quaternion Quaternion::Rotate(const Quaternion& quat, const Vector3& axis, double angle)
{
	return quat * Quaternion::FromAxisAngle(axis, angle);
}
double Quaternion::Dot(const Quaternion& quat1, const Quaternion& quat2)
{
	return (
		(quat1.GetX() * quat2.GetX())
		+ (quat1.GetY() * quat2.GetY())
		+ (quat1.GetZ() * quat2.GetZ())
		+ (quat1.GetW() * quat2.GetW())
	);
}
Quaternion Quaternion::LookAt(const Vector3& origin, const Vector3& target) {
	Vector3 forward = (target - origin).Normalized();
	double dot = Vector3::Dot(Vector3::Forward(), forward);

	if (std::abs(dot - (-1.0f)) < 0.000001f) {
		return Quaternion(
			Vector3::Up(),
			M_PI
		);
	}
	if (std::abs(dot - (1.0f)) < 0.000001f) {
		return Quaternion::Identity();
	}

	return FromAxisAngle(
		Vector3::Cross(Vector3::Forward(), forward).Normalized(),
		acos(dot)
	);
}
double Quaternion::Angle(const Quaternion& quat)
{
	return 2 * acos(quat.GetW());
}

// Methods
double Quaternion::Norm() const
{
	return Norm(*this);
}
double Quaternion::Length() const
{
	return Length(*this);
}
Quaternion Quaternion::Normalized() const
{
	return Normalized(*this);
}
Quaternion Quaternion::Normalize()
{
	return Normalize(*this);
}
Vector3 Quaternion::ToEulerAngles() const
{
	return ToEulerAngles(*this);
}
Vector3 Quaternion::GetAxis() const
{
	return GetAxis(*this);
}
Vector3 Quaternion::Rotate(const Vector3& vec) const
{
	return Rotate(*this, vec);
}
Quaternion Quaternion::Rotate(const Vector3& axis, double angle)
{
	return *this = Quaternion::Rotate(*this, axis, angle);
}
double Quaternion::Dot(const Quaternion& target) const
{
	return Dot(*this, target);
}
double Quaternion::Angle() const
{
	return Quaternion::Angle(*this);
}

// Operators
Quaternion Quaternion::operator=(const Quaternion& rh)
{
	m_X = rh.GetX();
	m_Y = rh.GetY();
	m_Z = rh.GetZ();
	m_W = rh.GetW();
	return *this;
}

Quaternion Quaternion::operator+(const Quaternion& rh) const
{
	return Quaternion(
		m_X + rh.GetX(),
		m_Y + rh.GetY(),
		m_Z + rh.GetZ(),
		m_W + rh.GetW()
	);
}
Quaternion Quaternion::operator+=(const Quaternion& rh)
{
	return *this = *this + rh;
}

Quaternion Quaternion::operator-(const Quaternion& rh) const
{
	return Quaternion(
		m_X - rh.GetX(),
		m_Y - rh.GetY(),
		m_Z - rh.GetZ(),
		m_W - rh.GetW()
	);
}
Quaternion Quaternion::operator-=(const Quaternion& rh)
{
	return *this = *this - rh;
}

Quaternion Quaternion::operator*(const Quaternion& rh) const
{
	double qX = rh.GetX();
	double qY = rh.GetY();
	double qZ = rh.GetZ();
	double qW = rh.GetW();
	return Normalize(
		Quaternion(
			(((m_W * qX) + (m_X * qW)) + (m_Y * qZ)) - (m_Z * qY),
			(((m_W * qY) + (m_Y * qW)) + (m_Z * qX)) - (m_X * qZ),
			(((m_W * qZ) + (m_Z * qW)) + (m_X * qY)) - (m_Y * qX),
			(((m_W * qW) - (m_X * qX)) - (m_Y * qY)) - (m_Z * qZ)
		)
	);
}
Quaternion Quaternion::operator*=(const Quaternion& rh)
{
	return *this = *this * rh;
}

Quaternion Quaternion::operator*(double rh) const
{
	return Quaternion(
		rh * GetX(),
		rh * GetY(),
		rh * GetZ(),
		rh * GetW()
	);
}
Quaternion Quaternion::operator*=(double rh)
{
	return *this = *this * rh;
}

Quaternion Quaternion::operator/(double rh) const
{
	return Quaternion(
		rh / GetX(),
		rh / GetY(),
		rh / GetZ(),
		rh / GetW()
	);
}
Quaternion Quaternion::operator/=(double rh)
{
	return *this = *this / rh;
}

Vector3 Quaternion::operator*(const Vector3& rh) const
{
	double num = m_X * 2.0f;
	double num2 = m_Y * 2.0f;
	double num3 = m_Z * 2.0f;
	double num4 = m_X * num;
	double num5 = m_Y * num2;
	double num6 = m_Z * num3;
	double num7 = m_X * num2;
	double num8 = m_X * num3;
	double num9 = m_Y * num3;
	double num10 = m_W * num;
	double num11 = m_W * num2;
	double num12 = m_W * num3;
	Vector3 result;
	result.SetX((1.0f - (num5 + num6)) * rh.GetX() + (num7 - num12) * rh.GetY() + (num8 + num11) * rh.GetZ());
	result.SetY((num7 + num12) * rh.GetX() + (1.0f - (num4 + num6)) * rh.GetY() + (num9 - num10) * rh.GetZ());
	result.SetZ((num8 - num11) * rh.GetX() + (num9 + num10) * rh.GetY() + (1.0f - (num4 + num5)) * rh.GetZ());
	return result;
}

bool Quaternion::operator==(const Quaternion& rh) const
{
	return (
		m_X == rh.GetX()
		&& m_Y == rh.GetY()
		&& m_Z == rh.GetZ()
		&& m_W == rh.GetW()
	);
}
bool Quaternion::operator!=(const Quaternion& rh) const
{
	return !(*this == rh);
}

Quaternion::operator std::string() const
{
	std::ostringstream strX;
	std::ostringstream strY;
	std::ostringstream strZ;
	std::ostringstream strW;
	strX << m_X;
	strY << m_Y;
	strZ << m_Z;
	strW << m_W;

	std::string retStr = "Quaternion(" +
		strX.str() + "f, " +
		strY.str() + "f, " +
		strZ.str() + "f, " +
		strW.str() + "f)";
	return retStr;
}
```
</details>

<details>
	<summary>src/networking/NetworkServer.hpp</summary>

```c++
#pragma once

#include <asio/ts/buffer.hpp>
#include <asio/ts/internet.hpp>

#include <cstdint>
#include <unordered_map>
#include <unordered_set>

class Component;
class Game;
class NetworkMessage;
template <typename T> class TDequeConcurrent;

typedef asio::ip::tcp::socket ConnectedClient;
typedef TDequeConcurrent<NetworkMessage*> ClientMessages;
typedef std::unordered_map<ConnectedClient*, ClientMessages*> NetworkMessageQueue;

class NetworkServer
{
	protected:
		uint16_t m_GameServerPort;
		const char* m_GameServerIp;
	public:
		static void HandleSessionReceiving(ConnectedClient*, ClientMessages*);
		static void HandleSessionSending(ConnectedClient*, ClientMessages*);
		static void HandleNewConnections(uint16_t, Game*);
		static std::unordered_set<ConnectedClient*>* GetConnectedClients();
		static NetworkMessageQueue* GetIncomingMessageQueue();
		static NetworkMessageQueue* GetOutgoingMessageQueue();
		static void PropagateOutgoingMessage(Component*, NetworkMessage*);

		// Propagate Outgoing message except for sender when not null
		static void PropagateOutgoingMessage(Component*, NetworkMessage*, ConnectedClient*);
	public:
		NetworkServer(const char*, uint16_t);
		void SetGameServerPort(uint16_t);
		void SetGameServerIp(const char*);
		void StartServer(Game*);
};

```
</details>

<details>
	<summary>src/networking/NetworkServer.cpp</summary>

```c++
#include <string>
#include <thread>

#include "../component/Component.hpp"
#include "../core/Logger.hpp"
#include "../Game.hpp"
#include "../maths/Vector3.hpp"
#include "../thread/TDequeConcurrent.hpp"
#include "NetworkMessage.hpp"
#include "NetworkMessageTransporter.hpp"
#include "NetworkMessageType.hpp"

#include "NetworkServer.hpp"

// For now, this really only re-broadcasts
// Server-authoritative-ness not implemented at all
// Sending client essentially proxies all messages through the server to all other clients
void NetworkServer::HandleSessionReceiving(ConnectedClient* client, ClientMessages* messageQueue)
{
	try {
		do {
			NetworkMessage* message = NetworkMessageTransporter::Receive(client);
			// Logger() << "Got message to rebroadcast: " << (std::string) *message << "\n";
			NetworkServer::PropagateOutgoingMessage(nullptr, message, client);
			delete message;
		} while (true);
	} catch (std::exception& e) {
		NetworkServer::GetConnectedClients()->erase(client);
		NetworkServer::GetOutgoingMessageQueue()->erase(client);
		Logger() << "Removed reference to connected client";
		Logger() << "Exception in thread: " << e.what() << "\n";
	}
}

void NetworkServer::HandleSessionSending(ConnectedClient* client, ClientMessages* messageQueue)
{
	try {
		do {
			NetworkMessage* messagePtr = messageQueue->pop_front();
			// Logger() << "Sending message: " << (std::string) *messagePtr << "\n";
			NetworkMessageTransporter::Send(client, messagePtr);

			delete messagePtr;
		} while (true);
	} catch (std::exception& e) {
		NetworkServer::GetConnectedClients()->erase(client);
		NetworkServer::GetOutgoingMessageQueue()->erase(client);
		Logger() << "Removed reference to connected client";
		Logger() << "Exception in thread: " << e.what() << "\n";
	}
}

void NetworkServer::HandleNewConnections(uint16_t port, Game* game)
{
	asio::io_context io_context;
	asio::ip::tcp::acceptor a(
		io_context,
		asio::ip::tcp::endpoint(asio::ip::tcp::v4(), port)
	);
	Logger() << "NetworkServer::HandleNewConnections: Ready to accept connections on port " << port << "\n";

	// TODO: Implement error handling in spawning threads and accepting connection
	// Shouldn't crash due to a failed connection with one client.
	do {
		ConnectedClient* sock = new ConnectedClient(io_context);
		a.accept(*sock);
		Logger() << "NetworkServer::HandleNewConnections: Accepting connection" << "\n";

		// Note that client has connected and establish outgoing/incoming message queue
		NetworkServer::GetConnectedClients()->insert(sock);
		ClientMessages* incomingMessageQueue = new ClientMessages();
		ClientMessages* outgoingMessageQueue = new ClientMessages();
		(*NetworkServer::GetOutgoingMessageQueue())[sock] = incomingMessageQueue;
		(*NetworkServer::GetOutgoingMessageQueue())[sock] = outgoingMessageQueue;

		// Spawn threads to handle incoming and outgoing messages
		std::thread(NetworkServer::HandleSessionSending, sock, outgoingMessageQueue).detach();
		std::thread(NetworkServer::HandleSessionReceiving, sock, incomingMessageQueue).detach();
	} while (game->IsRunning());
}

std::unordered_set<ConnectedClient*>* NetworkServer::GetConnectedClients()
{
	static std::unordered_set<ConnectedClient*>* s_ConnectedClients = new std::unordered_set<ConnectedClient*>();
	return s_ConnectedClients;
}

NetworkMessageQueue* NetworkServer::GetOutgoingMessageQueue()
{
	static NetworkMessageQueue* s_OutgoingMessageQueue = new NetworkMessageQueue();
	return s_OutgoingMessageQueue;
}

NetworkMessageQueue* NetworkServer::GetIncomingMessageQueue()
{
	static NetworkMessageQueue* s_IncomingMessageQueue = new NetworkMessageQueue();
	return s_IncomingMessageQueue;
}

void NetworkServer::PropagateOutgoingMessage(Component* component, NetworkMessage* message)
{
	PropagateOutgoingMessage(component, message, nullptr);
}

void NetworkServer::PropagateOutgoingMessage(Component* component, NetworkMessage* message, ConnectedClient* sender)
{
	// Add a to-be-sent message for every connected client
	NetworkMessageQueue* messageQueue = NetworkServer::GetOutgoingMessageQueue();

	for (auto connectedClient : *(NetworkServer::GetConnectedClients())) {
		// If we've been given a sender socket, ignore sending back to sender.
		// We are not USPS and we have no return to sender feature.
		if (sender != nullptr && sender == connectedClient) {
			// Logger() << "Skipping message to client, is originating sender" << "\n";
			continue;
		}
		NetworkMessage* clientMessage = new NetworkMessage(message);
		(*messageQueue)[connectedClient]->emplace_back(clientMessage);
	}
}

NetworkServer::NetworkServer(const char* ip, uint16_t port) :
	m_GameServerPort(port),
	m_GameServerIp(ip)
{
}

void NetworkServer::SetGameServerPort(uint16_t port)
{
	m_GameServerPort = port;
}

void NetworkServer::SetGameServerIp(const char* serverIp)
{
	m_GameServerIp = serverIp;
}

void NetworkServer::StartServer(Game* game)
{
	try {
		// Spawn thread to handle net server
		std::thread(NetworkServer::HandleNewConnections, m_GameServerPort, game).detach();
	} catch (std::exception& e) {
		Logger() << "NetworkServer::StartServer: Exception: " << (std::string) e.what() << "\n";
	}
}
```
</details>

<details>
	<summary>src/psp/PSPEventHandler.hpp</summary>

```c++
#pragma once

#include <unordered_map>

#include "../core/Input.hpp"
#include "../EventHandler.hpp"

class Game;

enum PressState {
	RELEASED,
	PRESSED,
};

class PSPEventHandler : public EventHandler
{
	private:
		Game* m_Game = nullptr;
		#ifdef __PSP__
			SceCtrlData* m_Pad = nullptr;
			const std::unordered_map<PspCtrlButtons, Input::Key> m_KeyPairs = {
				{PSP_CTRL_UP, Input::Key::KEY_UP},
				{PSP_CTRL_DOWN, Input::Key::KEY_DOWN},
				{PSP_CTRL_LEFT, Input::Key::KEY_LEFT},
				{PSP_CTRL_RIGHT, Input::Key::KEY_RIGHT},

				// Prototype action button mappings
				// TODO: Better this
				{PSP_CTRL_TRIANGLE, Input::Key::KEY_Y},
				{PSP_CTRL_CIRCLE, Input::Key::KEY_B},
				{PSP_CTRL_CROSS, Input::Key::KEY_A},
				{PSP_CTRL_SQUARE, Input::Key::KEY_X},
			};
		#endif
		std::unordered_map<Input::Key, PressState> m_LastKeyState;

	public:
		static PSPEventHandler* Self();

	public:
		#ifdef __PSP__
			void KeyCallback(PspCtrlButtons, int);
		#else
			void KeyCallback(int, int);
		#endif
		void SetGame(Game*);
		Game* GetGame();
		void Register();
		void PollEvents();
		void ResetCursor();
};

```
</details>

<details>
	<summary>src/psp/PSPEventHandler.cpp</summary>

```c++
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

#ifdef __PSP__
	#include <pspctrl.h>
#endif

#include "../core/Logger.hpp"
#include "../Game.hpp"

#include "PSPEventHandler.hpp"

PSPEventHandler* PSPEventHandler::Self()
{
	static PSPEventHandler* instance = new PSPEventHandler();
	return instance;
};

#ifdef __PSP__
	void PSPEventHandler::KeyCallback(PspCtrlButtons button, int action)
	{
		auto got = m_KeyPairs.find(button);
		if (got != m_KeyPairs.end()) {
			if (action == 1) {
				Logger() << "Key press: " << button << "\n";
				Input::Press(got->second);
			} else if (action == 0) {
				Logger() << "Key Release: " << button << "\n";
				Input::Release(got->second);
			}
		} else {
			Logger()
				<< "Unregistered button:" << button
				<< " action:" << action << "\n";
		}
	}
#else
	void PSPEventHandler::KeyCallback(int button, int action)
	{
		// Intentionally unimplemented
	}
#endif

void PSPEventHandler::SetGame(Game* game)
{
	m_Game = game;
}

Game* PSPEventHandler::GetGame()
{
	return m_Game;
}

void PSPEventHandler::Register()
{
	#ifdef __PSP__
		m_Pad = new SceCtrlData();
		sceCtrlSetSamplingCycle(0);
		sceCtrlSetSamplingMode(PSP_CTRL_MODE_ANALOG);

		// Set initial key states
		for (auto key : m_KeyPairs) {
			m_LastKeyState[key.second] = PressState::RELEASED;
		}
	#endif
}

void PSPEventHandler::PollEvents()
{
	#ifdef __PSP__
		sceCtrlReadBufferPositive(m_Pad, 1);
		for (auto keyPair : m_KeyPairs) {
			// Keys previously known as pressed
			if (m_LastKeyState[keyPair.second] == PressState::PRESSED) {
				// Not pressed anymore
				if (!(m_Pad->Buttons & keyPair.first)) {
					m_LastKeyState[keyPair.second] = PressState::RELEASED;
					Self()->KeyCallback(keyPair.first, 0);
				}

				// Do nothing with still pressed keys
				// We've already sent the notice

			// Newly pressed keys
			} else if (m_Pad->Buttons & keyPair.first) {
				m_LastKeyState[keyPair.second] = PressState::PRESSED;
				Self()->KeyCallback(keyPair.first, 1);
			}
		}
	#endif
}

void PSPEventHandler::ResetCursor()
{
}
```
</details>

<details>
	<summary>src/psp/PSPApplication.hpp</summary>

```c++
#pragma once

#include "../Application.hpp"

class EventHandler;
class Renderer;
class Window;
class Game;

class PSPApplication : public Application
{
	public:
		void Setup(Game*, EventHandler**, Renderer**, Window**);
};
```
</details>

<details>
	<summary>src/psp/PSPApplication.cpp</summary>

```c++
#ifdef __PSP__
	#include <pspkernel.h>
	#include <pspdebug.h>
	#include <pspctrl.h>
#endif

#include <stdlib.h>

#include "../EventHandler.hpp"
#include "../core/Logger.hpp"
#include "../Game.hpp"
#include "../graphics/psp/PSPWindow.hpp"
#include "../graphics/Renderer.hpp"
#include "../graphics/Window.hpp"
#include "PSPEventHandler.hpp"

#include "PSPApplication.hpp"

#ifdef __PSP__
	// TODO: This will likely need to be dynamic soonish
	PSP_MODULE_INFO("Game", 0, 1, 1);

	PSP_MAIN_THREAD_ATTR(THREAD_ATTR_USER | THREAD_ATTR_VFPU);
#endif

int exit_callback(int arg1, int arg2, void* common)
{
	PSPEventHandler::Self()->GetGame()->Stop();
    return 0;
}

#ifdef __PSP__
	int callback_thread(SceSize args, void* argp)
	{
		int cbid = sceKernelCreateCallback(
			"Exit Callback",
			exit_callback,
			nullptr
		);
		sceKernelRegisterExitCallback(cbid);
		sceKernelSleepThreadCB();
		return 0;
	}

	int setup_callbacks()
	{
		int thid = sceKernelCreateThread(
			"update_thread",
			callback_thread,
			0x11,
			0xFA0,
			0,
			0
		);

		if(thid >= 0) {
			sceKernelStartThread(thid, 0, 0);
		}
		return thid;
	}
#endif

void PSPApplication::Setup(Game* game, EventHandler** handler, Renderer** renderer, Window** window)
{
	#ifdef __PSP__
		// pspDebugScreenInit();
		setup_callbacks();
	#endif

	// Create window
	PSPWindow* newWindow = new PSPWindow(480, 272, "");

	// Create renderer
	Renderer* newRenderer = newWindow->GetRenderer();

	// Create & initialize event/input handlers
	PSPEventHandler* newHandler = PSPEventHandler::Self();
	newHandler->SetGame(game);
	// newHandler->SetWindow(newWindow->GetWindow());
	newHandler->Register();

	// Set the new data
	(*window) = newWindow;
	(*renderer) = newRenderer;
	(*handler) = newHandler;
}

```
</details>
</details>

## PSP-Pong

<details>
	<summary>Source Code</summary>
<details>
	<summary>CMakeLists.txt</summary>

```Makefile
cmake_minimum_required(VERSION 3.0)

project(pong)

set(
	ENGINE_SRC
	/mnt/c/Users/Brandon/Documents/Projects/cpp/onemonth/engine/src
)

SET (
	ENGINE_SRC_FILES
	
	# Component
	${ENGINE_SRC}/component/CameraComponent.cpp
	${ENGINE_SRC}/component/Component.cpp
	${ENGINE_SRC}/component/render/psp/2d/PSPRectRenderComponent.cpp
	${ENGINE_SRC}/component/render/RenderComponent.cpp
	${ENGINE_SRC}/component/Transform.cpp
	
	# Core
	${ENGINE_SRC}/core/Input.cpp
	${ENGINE_SRC}/core/Logger.cpp
	${ENGINE_SRC}/core/Timer.cpp

	# Graphics
	${ENGINE_SRC}/graphics/headless/HeadlessRenderer.cpp
	${ENGINE_SRC}/graphics/headless/HeadlessWindow.cpp
	${ENGINE_SRC}/graphics/psp/PSPRenderer.cpp
	${ENGINE_SRC}/graphics/psp/PSPWindow.cpp
	${ENGINE_SRC}/graphics/Renderer.cpp

	# Headless
	${ENGINE_SRC}/headless/HeadlessApplication.cpp
	${ENGINE_SRC}/headless/HeadlessEventHandler.cpp

	# Maths
	${ENGINE_SRC}/maths/Vector3.cpp
	${ENGINE_SRC}/maths/Quaternion.cpp

	# profiler
	${ENGINE_SRC}/profiler/Profiler.cpp

	# PSP
	${ENGINE_SRC}/psp/PSPApplication.cpp
	${ENGINE_SRC}/psp/PSPEventHandler.cpp

	${ENGINE_SRC}/Game.cpp
	${ENGINE_SRC}/GameLevel.cpp
	${ENGINE_SRC}/GameObject.cpp
)

set(
	SOURCES
	${ENGINE_SRC_FILES}
	src/main.cpp

	# Component
	src/component/BallControllerComponent.cpp
	src/component/OpponentPaddleControllerComponent.cpp
	src/component/PlayerPaddleControllerComponent.cpp

	# Level
	src/level/TestGameLevel.cpp

	# Preset
	src/preset/CameraPreset.cpp
	src/preset/BallPreset.cpp
	src/preset/PaddlePreset.cpp
	src/preset/OpponentPaddlePreset.cpp
	src/preset/PlayerPaddlePreset.cpp
)

add_executable(${PROJECT_NAME} ${SOURCES})

target_include_directories(${PROJECT_NAME} PRIVATE
	${ENGINE_SRC}
)

target_link_libraries(${PROJECT_NAME} PRIVATE
	pspctrl
	pspdebug
	pspdisplay
	pspge
	pspgu
)

# Create an EBOOT.PBP file
create_pbp_file(
	TARGET ${PROJECT_NAME}
	ICON_PATH NULL
	BACKGROUND_PATH NULL
	PREVIEW_PATH NULL
	TITLE ${PROJECT_NAME}
)
```
</details>

<details>
	<summary>src/main.cpp</summary>

```c++
#include <core/Logger.hpp>
#include <Game.hpp>
#include <GameMode.hpp>

#include "./level/TestGameLevel.hpp"

int main(int argc, char* args[]) 
{
	Logger::Silence();
	GameMode mode = GameMode::CLIENT;
	Game game(mode);
	game.UpdateTickRate(60);

	TestGameLevel* level = new TestGameLevel();
	game.ChangeToLevel(level);
	game.Run();

	return 0;
}
```
</details>

<details>
	<summary>src/component/BallControllerComponent.hpp</summary>

```c++
#pragma once

// Engine
#include "maths/Vector3.hpp"

#include <component/Component.hpp>

class BallControllerComponent : public Component
{
	private:
		double m_TravelSpeed = 384.0f;
		Vector3 m_TravelVector;
		float m_DelayMovement = 0.75f;

	public:
		void Start();
		void Update(double deltaTime);
		void ResetPosition();
};
```
</details>

<details>
	<summary>src/component/BallControllerComponent.cpp</summary>

```c++
// Engine
#include "component/Transform.hpp"
#include "core/Collection.hpp"
#include "core/Input.hpp"
#include "Game.hpp"
#include "GameLevel.hpp"
#include "GameObject.hpp"
#include "maths/Vector3.hpp"

#include "PlayerPaddleControllerComponent.hpp"

#include "BallControllerComponent.hpp"

// https://stackoverflow.com/a/865287
struct Rect
{
	float x;
	float y;
	float width;
	float height;
};
bool rectsCollide(Rect r1, Rect r2)
{
	if (r1.x + r1.width < r2.x) return false; // r2 too far right
	if (r1.x > r2.x + r2.width) return false; // r2 too far left
	if (r1.y > r2.y + r2.height) return false; // r2 too high
	if (r1.y + r1.height < r2.y) return false; // r2 too low
	return true;
}

void BallControllerComponent::ResetPosition()
{
	m_Transform->SetPosition(Vector3(232.0f, 120.0f, 0.0f));

	// Add a ball-move delay
	m_DelayMovement = 0.75;
}

void BallControllerComponent::Start()
{
	m_TravelVector = Vector3(Vector3::Right() * -1);
	ResetPosition();
}

void BallControllerComponent::Update(double deltaTime)
{
	if (m_DelayMovement > 0) {
		m_DelayMovement -= (float) deltaTime;
		return;
	}

	Vector3* position = m_Transform->GetPosition();
	double speed = m_TravelSpeed * deltaTime;

	// Move
	*position += m_TravelVector * speed;

	// Collision checks
	Rect ballRect = {
		x: (float) m_Transform->GetPosition()->GetX(),
		y: (float) m_Transform->GetPosition()->GetY(),
		width: (float) m_Transform->GetScale()->GetX(),
		height: (float) m_Transform->GetScale()->GetY(),
	};

	GameObject* player1 = m_GameObject->GetLevel()->GetObjectByName("Opponent");
	Rect p1Rect = {
		x: (float) player1->GetTransform()->GetPosition()->GetX(),
		y: (float) player1->GetTransform()->GetPosition()->GetY(),
		width: (float) player1->GetTransform()->GetScale()->GetX(),
		height: (float) player1->GetTransform()->GetScale()->GetY(),
	};

	GameObject* player2 = m_GameObject->GetLevel()->GetObjectByName("Player");
	Rect p2Rect = {
		x: (float) player2->GetTransform()->GetPosition()->GetX(),
		y: (float) player2->GetTransform()->GetPosition()->GetY(),
		width: (float) player2->GetTransform()->GetScale()->GetX(),
		height: (float) player2->GetTransform()->GetScale()->GetY(),
	};

	// If hit either paddle
	// left player
	if (rectsCollide(ballRect, p1Rect)) {
		// invert travel vector
		m_TravelVector *= -1;

		// Prevent intersection rapid bouncing
		position->SetX(p1Rect.x + p1Rect.width);

	// right player
	} else if (rectsCollide(ballRect, p2Rect)) {
		// invert travel vector
		m_TravelVector *= -1;
		Vector3 p2CurPos = *(player2->GetTransform()->GetPosition());
		Vector3 p2PrevPos = player2
			->Components()
			->Get<PlayerPaddleControllerComponent>()
			->GetPreviousPosition();
		if (p2CurPos != p2PrevPos) {
			Vector3 modVec = Vector3::Up();
			if ((p2CurPos - p2PrevPos).GetY() < 0) {
				modVec *= -1;
			}
			m_TravelVector = ((modVec + m_TravelVector) * modVec.Angle(m_TravelVector)).Normalized();
		}

		// Prevent intersection rapid bouncing
		position->SetX(p2Rect.x - ballRect.width);
	}

	// If out of left-right bounds, reset
	if (ballRect.x <= 0 || ballRect.x + ballRect.width >= 480) {
		Vector3 newTravelVec = Vector3::Right();
		if (ballRect.x <= 0) {
			newTravelVec *= -1;
		}
		m_TravelVector = newTravelVec;
		ResetPosition();
		return;
	}


	// If out of top-bottom bounds, bounce
	if (ballRect.y < 0) {
		// Deflect off ceiling
		m_TravelVector.SetY(m_TravelVector.GetY() * -1);
		position->SetY(1);

	} else if (ballRect.y + ballRect.height > 272) {
		// Deflect off floor
		m_TravelVector.SetY(m_TravelVector.GetY() * -1);
		position->SetY(272 - ballRect.height - 1);
	}
}
```
</details>

<details>
	<summary>src/component/OpponentPaddleControllerComponent.hpp</summary>

```c++
#pragma once

// Engine
#include "maths/Vector3.hpp"

#include <component/Component.hpp>

class OpponentPaddleControllerComponent : public Component
{
	private:
		double m_TravelSpeed = 192.0f;
		float m_ChangeDirDelay = 0;
		Vector3 m_TravelVector = Vector3::Up();
		Vector3 m_NewTravelVector = Vector3::Up();
		Vector3 m_PreviousPosition;

	public:
		void Start();
		void Update(double deltaTime);
		const Vector3 GetPreviousPosition();
};
```
</details>

<details>
	<summary>src/component/OpponentPaddleControllerComponent.cpp</summary>

```c++
// Engine
#include "component/Transform.hpp"
#include "core/Input.hpp"
#include "core/Logger.hpp"
#include "Game.hpp"
#include "GameLevel.hpp"
#include "GameObject.hpp"
#include "maths/Vector3.hpp"

#include "OpponentPaddleControllerComponent.hpp"

void OpponentPaddleControllerComponent::Start()
{
	m_PreviousPosition = *m_Transform->GetPosition();
}

void OpponentPaddleControllerComponent::Update(double deltaTime)
{
	Vector3* position = m_Transform->GetPosition();
	Vector3* scale = m_Transform->GetScale();
	float curYCenter = position->GetY() - (scale->GetY() / 2);
	Vector3* ballPos = m_GameObject
		->GetLevel()
		->GetObjectByName("Ball")
		->GetTransform()
		->GetPosition();
	Vector3* ballScale = m_GameObject
		->GetLevel()
		->GetObjectByName("Ball")
		->GetTransform()
		->GetScale();
	float ballYCenter = ballPos->GetY() + (ballScale->GetY() / 2);

	// We'll use this to create a deflection angle for the ball on collision
	m_PreviousPosition = *position;

	// Move to match ball within some boundary
	// TODO: Some semi-intelligent AI
	double deltaPos = m_TravelSpeed * deltaTime;
	if (
		ballYCenter < position->GetY()
		|| ballYCenter > position->GetY() + scale->GetY()
	) {
		m_TravelVector = Vector3::Up();
		if (ballYCenter > position->GetY() + scale->GetY()) {
			m_TravelVector *= -1;
		}
		*position -= m_TravelVector * deltaPos;
	}

	// Clamp position within screen
	// TODO: Get this from Game->Window->Renderer/whatever
	if (position->GetY() > 203) {
		position->SetY(203);
	} else if (position->GetY() < 5) {
		position->SetY(5);
	}
}

const Vector3 OpponentPaddleControllerComponent::GetPreviousPosition()
{
	return m_PreviousPosition;
}
```
</details>

<details>
	<summary>src/component/PlayerPaddleControllerComponent.hpp</summary>

```c++
#pragma once

// Engine
#include "maths/Vector3.hpp"

#include <component/Component.hpp>

class PlayerPaddleControllerComponent : public Component
{
	private:
		double m_TravelSpeed = 246.0f;
		Vector3 m_PreviousPosition;

	public:
		void Start();
		void Update(double deltaTime);
		const Vector3 GetPreviousPosition();
};
```
</details>

<details>
	<summary>src/component/PlayerPaddleControllerComponent.cpp</summary>

```c++
// Engine
#include "component/Transform.hpp"
#include "core/Input.hpp"
#include "Game.hpp"
#include "GameLevel.hpp"
#include "GameObject.hpp"
#include "maths/Vector3.hpp"

#include "PlayerPaddleControllerComponent.hpp"

void PlayerPaddleControllerComponent::Start()
{
	m_PreviousPosition = *m_Transform->GetPosition();
}

void PlayerPaddleControllerComponent::Update(double deltaTime)
{
	Vector3* position = m_Transform->GetPosition();

	// We'll use this to create a deflection angle for the ball on collision
	m_PreviousPosition = *position;

	// For now, this will be our exit button
	if (Input::Pressed(Input::Key::KEY_B)) {
		m_GameObject->GetLevel()->GetGame()->Stop();
	}

	// Move
	if (
		Input::Pressed(Input::Key::KEY_UP)
		|| Input::Pressed(Input::Key::KEY_DOWN)
	) {
		double deltaPos = m_TravelSpeed * deltaTime;
		Vector3 localUp = Vector3::Up();

		// Move Up/Down
		// TODO: Unsure why this is inverted
		if (Input::Pressed(Input::Key::KEY_UP)) {
			*position -= localUp * deltaPos;
		}
		if (Input::Pressed(Input::Key::KEY_DOWN)) {
			*position += localUp * deltaPos;
		}

		// Clamp position within screen
		// TODO: Get this from Game->Window->Renderer/whatever
		if (position->GetY() > 203) {
			position->SetY(203);
		} else if (position->GetY() < 5) {
			position->SetY(5);
		}
	}
}

const Vector3 PlayerPaddleControllerComponent::GetPreviousPosition()
{
	return m_PreviousPosition;
}
```
</details>
</details>
