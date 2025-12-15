# ClassicsArcade

## Introduction

&emsp;ClassicsArcade is a Unity-based collection of classic arcade game remakes, featuring multiple games developed by a team of developers. The project showcases modern game development practices while paying homage to the golden age of arcade gaming. The featured game, **Typeroids**, is a unique twist on the classic Asteroids formula that combines typing skills with fast-paced action gameplay.

&emsp;In Typeroids, asteroids drift toward the player's ship at the center of the screen, each displaying a word. Players must type the words as quickly as possible to fire projectiles and destroy the asteroids before they collide with the ship. The game features progressive difficulty scaling, a scoring system that rewards skill and risk-taking, and wave-based gameplay that keeps the challenge fresh.

&emsp;Below you'll find detailed descriptions of the core systems and features, along with code snippets and implementation details. The project demonstrates object-oriented design patterns, manager-based architecture, and efficient game systems built for performance and maintainability.

## Featured Game: Typeroids

*Jump To: [Core Systems](#core-systems), [Gameplay Features](#gameplay-features), [Architecture](#architecture), [Technical Highlights](#technical-highlights)*

### Core Systems

#### Typing System

&emsp;The heart of Typeroids is the typing mechanics. Unlike traditional Asteroids where you aim and shoot, players must type the words displayed on each asteroid. The system features intelligent auto-targeting that automatically selects the best asteroid based on what letter the player types, prioritizing proximity to the ship and word length.

```csharp
// Processes a single letter input from the keyboard
private void ProcessLetterInput(char inputChar)
{
    char upperChar = ignoreCase ? char.ToUpper(inputChar) : inputChar;
    Asteroid previousTarget = currentTarget;
    
    // Auto-targeting: Find asteroid with matching next letter
    if (currentTarget == null || !currentTarget.IsActive || 
        currentTarget.GetNextLetter() != upperChar)
    {
        // Find an asteroid whose next letter matches what we typed
        currentTarget = asteroidManager.GetAsteroidByNextLetter(upperChar);
        
        if (currentTarget != null)
        {
            currentTarget.SetTargeted(true); // Highlights next letter
        }
    }
    
    // Update progress and fire projectile when word is complete
    if (nextRequiredLetter == upperChar)
    {
        OnCorrectInput(currentTarget, upperChar);
    }
}
```

&emsp;The typing system tracks progress for each asteroid independently, using color-coded visual feedback (green for typed letters, yellow for the next letter to type) to give players clear indication of their progress. When a word is completed, a projectile automatically fires at that asteroid, seamlessly combining typing mechanics with traditional shooting gameplay.

#### Asteroid Management

&emsp;Asteroids spawn from the edges of the screen and drift inward toward the player. The system dynamically adjusts difficulty based on wave progression, spawning more asteroids with longer words and faster speeds as the game progresses.

```csharp
// Calculates how many asteroids to spawn based on wave number
private int CalculateAsteroidsForWave(int wave)
{
    // Each wave adds about 1.5 more asteroids
    return baseAsteroidsPerWave + Mathf.RoundToInt((wave - 1) * 1.5f);
}

// Determines asteroid size distribution based on wave
private AsteroidSize DetermineAsteroidSize(int wave)
{
    float rand = Random.Range(0f, 1f);
    
    if (wave <= 2)
    {
        // Waves 1-2: Mostly large asteroids (easier)
        return rand < 0.8f ? AsteroidSize.Large : AsteroidSize.Medium;
    }
    else if (wave <= 5)
    {
        // Waves 3-5: Mix of all sizes
        if (rand < 0.5f) return AsteroidSize.Large;
        if (rand < 0.8f) return AsteroidSize.Medium;
        return AsteroidSize.Small;
    }
    else
    {
        // Wave 6+: More small asteroids (harder!)
        if (rand < 0.3f) return AsteroidSize.Large;
        if (rand < 0.6f) return AsteroidSize.Medium;
        return AsteroidSize.Small;
    }
}
```

&emsp;Smaller asteroids move faster and display longer words, making them more challenging targets. The scoring system rewards players with bonus multipliers for destroying smaller asteroids and completing longer words, creating risk/reward gameplay where players can choose to prioritize easier targets or go for higher-scoring challenges.

#### Collision Detection

&emsp;Rather than relying solely on Unity's physics system, the collision detection uses a custom distance-based approach that gives precise control over hit detection. This allows for smooth, frame-rate independent collision checking while maintaining performance.

```csharp
// Checks if any asteroids are colliding with the ship
private void CheckCollisions()
{
    if (isInvincible) return; // Skip if player has temporary invincibility
    
    var activeAsteroids = asteroidManager.GetActiveAsteroids();
    Vector3 shipPos = shipTransform.position;
    
    foreach (var asteroid in activeAsteroids)
    {
        if (asteroid == null || !asteroid.IsActive) continue;
        
        float distance = Vector3.Distance(shipPos, asteroid.transform.position);
        float asteroidRadius = asteroid.GetCollisionRadius();
        
        // Collision if distance is less than combined radii
        if (distance < (collisionRadius + asteroidRadius))
        {
            OnCollision(asteroid);
            break; // Only process one collision per frame
        }
    }
}
```

&emsp;After a collision, the player receives a brief invincibility period with visual feedback (flashing red) to prevent instant death from multiple simultaneous hits. This creates a forgiving but challenging gameplay experience.

*Jump To: [Introduction](#introduction), [Gameplay Features](#gameplay-features), [Architecture](#architecture), [Technical Highlights](#technical-highlights)*

### Gameplay Features

#### Wave Progression

&emsp;The game uses a wave-based progression system where each wave increases in difficulty. When all asteroids in a wave are destroyed, the game transitions to the next wave with a brief delay, spawning a new set of asteroids. The wave number affects:

- **Asteroid count**: More asteroids spawn in later waves
- **Word length**: Longer words appear on asteroids as waves progress
- **Asteroid speed**: Faster movement toward the center
- **Size distribution**: More small, fast-moving asteroids in later waves
- **Scoring multipliers**: Higher wave numbers provide score bonuses

```csharp
// Moves to the next wave - gets harder each time!
public void StartNextWave()
{
    if (gameState != GameState.Playing) return;
    
    currentWave++;
    OnWaveChanged?.Invoke(currentWave);
    
    if (uiManager != null)
    {
        uiManager.UpdateWave(currentWave);
        uiManager.ShowWaveIntro(currentWave); // "Wave X" message
    }
    
    // Tell AsteroidManager to spawn asteroids for this wave
    if (asteroidManager != null)
    {
        asteroidManager.SpawnWave(currentWave);
    }
}
```

#### Scoring System

&emsp;The scoring system rewards skillful play through multiple multipliers. Points are calculated based on word length, asteroid size, and current wave, encouraging players to take on challenging targets for maximum points.

```csharp
// Scoring rewards: longer words, smaller asteroids, later waves
public void OnWordCompleted(int wordLength, AsteroidSize size, int wave)
{
    wordsCompleted++;
    
    float sizeMultiplier = GetSizeMultiplier(size);
    float waveMultiplier = GetWaveMultiplier(wave);
    int points = Mathf.RoundToInt(wordLength * sizeMultiplier * waveMultiplier);
    
    AddScore(points);
}

private float GetSizeMultiplier(AsteroidSize size)
{
    switch (size)
    {
        case AsteroidSize.Large: return 1.0f;   // Base points
        case AsteroidSize.Medium: return 1.5f;   // 50% bonus
        case AsteroidSize.Small: return 2.0f;    // Double points!
        default: return 1.0f;
    }
}

private float GetWaveMultiplier(int wave)
{
    // Wave 1: 1.0, Wave 2: 1.2, Wave 3: 1.4, etc.
    return 1.0f + (wave - 1) * 0.2f;
}
```

&emsp;High scores are persisted using Unity's PlayerPrefs system, allowing players to track their best performance across game sessions.

#### Lives and Game Over

&emsp;Players start with a set number of lives (default: 3). Each asteroid collision removes one life. When lives reach zero, the game transitions to the game over screen, displaying final statistics including score, waves survived, and words typed.

```csharp
// Called when an asteroid collides with the player ship
public void OnAsteroidHitPlayer()
{
    if (gameState != GameState.Playing) return;
    
    currentLives--;
    OnLivesChanged?.Invoke(currentLives);
    
    if (uiManager != null)
    {
        uiManager.UpdateLives(currentLives);
    }
    
    // Check if game over
    if (currentLives <= 0)
    {
        GameOver();
    }
}
```

*Jump To: [Introduction](#introduction), [Core Systems](#core-systems), [Architecture](#architecture), [Technical Highlights](#technical-highlights)*

### Architecture

#### Manager-Based Design

&emsp;The game uses a manager-based architecture where different systems are handled by specialized manager classes. This separation of concerns makes the codebase maintainable and allows for easy extension of features.

- **GameManager**: Central game state manager using the Singleton pattern. Handles score, lives, waves, and game flow.
- **AsteroidManager**: Manages asteroid spawning, word assignment, and wave progression.
- **TypingSystem**: Handles keyboard input and word completion tracking.
- **UIManager**: Manages all UI elements, menus, and HUD updates.
- **CollisionSystem**: Custom collision detection between asteroids and the player ship.

#### Singleton Pattern

&emsp;The GameManager uses the Singleton pattern to provide global access to game state while preventing duplicate instances. This allows other systems to easily access game state without tight coupling.

```csharp
public static GameManager Instance { get; private set; }

private void Awake()
{
    if (Instance == null)
    {
        Instance = this;
        DontDestroyOnLoad(gameObject); // Persist across scenes
    }
    else
    {
        Destroy(gameObject); // Prevent duplicates
        return;
    }
}
```

#### Event System

&emsp;The game uses C# events to create loose coupling between systems. When game state changes (score updates, lives lost, wave progression), events notify subscribed systems without requiring direct references.

```csharp
// Events that other systems can subscribe to
public System.Action<int> OnScoreChanged;
public System.Action<int> OnLivesChanged;
public System.Action<int> OnWaveChanged;
public System.Action OnGameOver;
public System.Action OnGameStart;

// Example: UIManager subscribes in Start()
GameManager.Instance.OnGameStart += OnGameStarted;
GameManager.Instance.OnGameOver += OnGameOver;
```

#### Scene Management

&emsp;The project includes a reusable SceneLoader helper class that simplifies scene transitions. This utility is used across the arcade collection for consistent scene loading behavior.

```csharp
public class SceneLoader : MonoBehaviour
{
    public void LoadScene(string sceneName)
    {
        SceneManager.LoadScene(sceneName);
    }
    
    public void LoadScene(int sceneIndex)
    {
        SceneManager.LoadScene(sceneIndex);
    }
    
    public void ReloadCurrentScene()
    {
        SceneManager.LoadScene(SceneManager.GetActiveScene().buildIndex);
    }
}
```

*Jump To: [Introduction](#introduction), [Core Systems](#core-systems), [Gameplay Features](#gameplay-features), [Technical Highlights](#technical-highlights)*

### Technical Highlights

#### Object Pooling Infrastructure

&emsp;The project includes an ObjectPool utility class designed for efficient object reuse. While not fully implemented in the current version, the infrastructure is in place for optimizing projectile and asteroid spawning.

```csharp
public class ObjectPool : MonoBehaviour
{
    private Queue<GameObject> availableObjects = new Queue<GameObject>();
    
    public GameObject GetPooledObject()
    {
        if (availableObjects.Count > 0)
        {
            GameObject obj = availableObjects.Dequeue();
            obj.SetActive(true);
            return obj;
        }
        
        if (expandPool)
        {
            return CreatePooledObject();
        }
        
        return null;
    }
    
    public void ReturnToPool(GameObject obj)
    {
        if (obj != null && allObjects.Contains(obj))
        {
            obj.SetActive(false);
            obj.transform.SetParent(transform);
            availableObjects.Enqueue(obj);
        }
    }
}
```

#### Visual Feedback Systems

&emsp;Asteroids provide rich visual feedback through TextMeshPro color coding:

- **White**: Letters not yet typed
- **Green**: Letters successfully typed
- **Yellow**: Next letter to type (when asteroid is targeted)

This color coding helps players quickly identify their progress and priorities.

```csharp
// Updates text display with color coding
private void UpdateTextDisplay()
{
    string displayText = "";
    
    for (int i = 0; i < word.Length; i++)
    {
        char c = word[i];
        
        if (i < typedProgress.Length)
        {
            // Typed letters in green
            displayText += $"<color=#{ColorUtility.ToHtmlStringRGB(typedTextColor)}>{c}</color>";
        }
        else if (isTargeted && i == typedProgress.Length)
        {
            // Next letter to type in yellow
            displayText += $"<color=#{ColorUtility.ToHtmlStringRGB(targetColor)}>{c}</color>";
        }
        else
        {
            displayText += c; // Future letters stay white
        }
    }
    
    textMesh.text = displayText;
}
```

#### Namespace Organization

&emsp;The codebase uses C# namespaces to organize code logically:

- `Typeroids.Managers`: Core game management systems
- `Typeroids.Gameplay`: Gameplay entities (PlayerShip, Asteroid, Projectile)
- `Typeroids.Data`: Data structures and enums (GameState, AsteroidSize)
- `Typeroids.Utilities`: Utility classes (ObjectPool, ScreenBounds)
- `Typeroids.Helpers`: Helper classes (SceneLoader, AudioManager)

This organization keeps the codebase clean and makes it easy to understand the relationship between different components.

#### Pause System

&emsp;The game implements a pause system using Unity's Time.timeScale, allowing players to pause and resume gameplay. The pause state is integrated into the game state management system.

```csharp
public void PauseGame()
{
    if (gameState == GameState.Playing)
    {
        gameState = GameState.Paused;
        Time.timeScale = 0f; // Stop all time-based operations
    }
}

public void ResumeGame()
{
    if (gameState == GameState.Paused)
    {
        gameState = GameState.Playing;
        Time.timeScale = 1f; // Resume normal speed
    }
}
```

*Jump To: [Introduction](#introduction), [Core Systems](#core-systems), [Gameplay Features](#gameplay-features), [Architecture](#architecture)*

## Project Structure

### Game Collection

&emsp;ClassicsArcade is designed as a collection of multiple classic arcade games. In addition to Typeroids, the project includes games in various stages of development:

- **Space Invaders** (multiple variations)
- **Pac-Man**
- **Galaga**
- **Frogger**
- **Missile Command**
- **Donkey Kong**
- **1942**
- **Joust**
- **Flappy Bird**
- And more...

Each game has its own folder structure while sharing common utilities and the main menu system.

### Shared Systems

&emsp;The project includes a `Common` folder for shared assets and utilities used across multiple games. The `ClassicArcade` folder contains the main menu system that serves as the hub for accessing different games in the collection.

### Development Team

&emsp;This is a collaborative project with multiple developers contributing individual games to the collection. The modular architecture allows each developer to work independently while maintaining consistency through shared utilities and design patterns.

## Key Technologies

- **Unity Engine**: 2D game development
- **C#**: Primary programming language
- **TextMeshPro**: Advanced text rendering for word display
- **Unity UI System**: Menu and HUD implementation
- **Unity Physics2D**: Collision detection foundation

## Code Philosophy

&emsp;The codebase emphasizes readability and maintainability through:

- **Comprehensive comments**: Code is well-documented with inline explanations
- **Clear naming conventions**: Variables and methods use descriptive names
- **Separation of concerns**: Each class has a single, well-defined responsibility
- **Event-driven architecture**: Loose coupling between systems
- **Extensible design**: Patterns and utilities designed for reuse

&emsp;This makes the codebase approachable for both experienced developers and those learning game development, with each system clearly explaining its purpose and implementation details.

## Future Enhancements

Potential areas for expansion and improvement:

- Full object pooling implementation for projectiles and asteroids
- Particle effects and visual polish
- Sound effects and background music
- Power-ups and special abilities
- Leaderboard system
- Additional word lists and difficulty modes
- Mobile platform support

---

*This project demonstrates modern Unity game development practices while creating an engaging, skill-based gameplay experience that combines typing proficiency with classic arcade action.*

