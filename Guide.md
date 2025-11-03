# Build Intelligent NPCs with Agora's ConvoAI Engine in Unity

If you've shipped a game with NPCs, you know the disconnect players feel when voice acting loops, dialogue trees hit dead ends, or characters can't respond to anything outside their scripted paths. Players expect conversations, not multiple choice quizzes.

Voice AI changes this. Instead of pre-recording every possible response or building complex branching logic, you can create NPCs that actually understand what players say and respond naturally. The technical challenge isn't the AI itself anymore—it's integrating it into your game loop without tanking performance or adding network latency that breaks immersion.

That's where Agora's ConvoAI Engine comes in. It handles the real-time audio processing, speech recognition, and LLM orchestration while you focus on game logic and character behavior. The same WebRTC infrastructure that powers low-latency video calls at scale works for voice-driven NPCs—sub-300ms response times, even in multiplayer scenarios.

This guide walks through implementing intelligent NPCs in Unity: from basic setup and voice capture to managing conversation state and optimizing for production. By the end, you'll have NPCs that players can actually talk to, not just talk at.

## Understand the tech

Agora's Conversational AI Engine treats NPCs as participants in your game's audio channels—just like multiplayer voice chat, but isntead of two human users one is AI.

This means you're not bolting on a separate chatbot API; you're extending the real-time communication layer you may already be using.

Your Unity client captures player voice through Agora's RTC SDK (the same one handling multiplayer voice). When a player speaks to an NPC, that audio stream gets routed to Conversational AI Engine's backend, where three things happen in parallel:

1. ASR (Automatic Speech Recognition) transcribes the audio to text—handles accents, background noise, and partial utterances without you writing audio processing code
2. LLM processing takes that transcript plus your NPC's personality config (more on this later) and generates a contextual response
3. TTS (Text-to-Speech) converts the AI response back to audio with voice characteristics you define.

To implement conversational AI NPCs with Agora, you need to understand a few key concepts.

1. Your Unity game initializes Agora's RTC Client SDK
2. Make a request to Agora's Conversational AI Engine to create an AI agent (NPC Agent)
3. The NPC Agent joins the same audio/video channels as players using unique agent IDs
4. The Conversational AI Engine processes player voice input through ASR (speech-to-text) and passes the transcript to the LLM
5. LLM generates contextual responses based on NPC personality configuration
6. TTS converts AI responses to natural speech delivered through Agora's network back to the user.
7. The Agora RTC SDK manages all real-time communication complexities including network optimization and quality adaptation

The following figure shows the data flow between your Unity application, players, and Agora's ConvoAI infrastructure:

![Conversational AI Architecture](https://github.com/user-attachments/assets/b1ed62ed-bd33-4e54-bfde-4d915fec0eb9)

The Conversational AIled Engine seamlessly integrates with Agora's proven Voice SDK, allowing NPCs to participate naturally in player conversations while maintaining ultra-low latency and enterprise-grade reliability.

## Prerequisites

To successfully implement conversational AI NPCs with Agora, you must have the following:

- A valid Agora account. If you don't have an account, see how to [Get started with Agora](https://docs.agora.io/en/Agora%20Platform/get_appid_token?platform=All%20Platforms&utm_source=medium&utm_medium=blog&utm_campaign=Build_Intelligent_NPCs_with_Agora_ConvoAI_Engine_in_Unity).
- An Agora project with Conversational AI Engine access enabled.
- Unity 2020.3 or later with the Agora RTC SDK for Unity installed.
- Basic knowledge of C# and Unity development.
- API credentials for your chosen LLM provider (OpenAI, Azure OpenAI, etc.).
- TTS service credentials (Azure Cognitive Services, Amazon Polly, etc.).

## Project setup

The ConvoAI Unity package wraps Agora's RTC SDK with AI-specific components, so you're setting up both real-time audio infrastructure and the AI agent layer. Here's how to wire it up:

### 1. Pull down the integration package

Clone the [AgoraConvoAI_Unity repository](https://github.com/chitimalli/AgoraConvoAI_Unity) into your Unity project's root. This gives you:

- Pre-configured Agora RTC SDK (v4.x)
- ConvoAI wrapper scripts
- Example NPC implementations you can reference

Import the Agora RTC SDK from `Assets/Agora-RTC-Plugin/`. If Unity throws dependency warnings, you're likely missing iOS/Android build support modules—you can ignore these unless you're targeting mobile.

### 2. Create your ConvoAI config asset

Right-click in your Project window → **Create → Agora → ConvoAIConfigs**. Name it `ConvoAIConfigs` (singular, despite what the menu says).

This ScriptableObject holds your Agora App ID, NPC personality definitions, and channel configuration. You'll come back to this in the next section—for now, just create it so the scripts can reference it.

### 3. Structure your scene

Here's the minimal setup I use for testing:

```
Scene Hierarchy:
├── ConvoAI Manager (Empty GameObject)
│   └── JoinConvoAIChannelAudio (Component)
├── NPC Character (3D model)
│   └── GameNPC (Custom Script)
└── UI Canvas
    ├── Voice Input Indicator
    └── NPC Status Display (helpful for debugging)
```

**What each piece does:**

- **ConvoAI Manager**: Initializes the Agora RTC engine and manages the audio channel lifecycle. This is your singleton—only one per scene.
- **JoinConvoAIChannelAudio**: Handles joining/leaving channels and routing audio between the player and AI agents. Attach this to the Manager GameObject.
- **GameNPC**: Your custom script that defines NPC behavior—personality, dialogue triggers, game state integration. You'll write this.
- **Voice Input Indicator**: Visual feedback so players know the game is listening. Even a simple pulsing circle works.

## Build

Implementing conversational AI NPCs with Agora involves configuring the ConvoAI system and creating intelligent NPC behaviors. This section shows you how to:

- [Initialize the ConvoAI system and create AI agents](#initialize-convoai-system)
- [Configure NPC personalities and behaviors](#configure-npc-personalities)
- [Handle voice interactions and responses](#handle-voice-interactions)

### Initialize ConvoAI system

Now we wire up the actual AI agent. This involves three layers: configuration (what your NPC knows), initialization (connecting to Agora's infrastructure), and status handling (reacting to the AI's state).

1. Configure your AI agent  
   The ConvoAIConfigs ScriptableObject is where you define both the technical plumbing and your NPC's personality.

   ```csharp
   [CreateAssetMenu(fileName = "ConvoAIConfigs", menuName = "Agora/ConvoAIConfigs")]
   public class ConvoAIConfigs : ScriptableObject
   {
       [Header("Agora RTC Configuration")]
       public string appId;        // Your Agora App ID
       public string token;        // RTC token for authentication
       public string channelName;  // Channel where NPC will join

       [Header("ConvoAI API Configuration")]
       public string apiKey;       // ConvoAI API key
       public string apiSecret;    // ConvoAI API secret
       public string agentName;    // Custom name for your AI agent
       public uint agentRtcUid = 8888; // Unique ID for the agent

       [Header("AI Personality Configuration")]
       public string systemMessage;   // Defines NPC personality and behavior
       public string greetingMessage; // Initial greeting when activated
       public string failureMessage;  // Fallback response for errors
       public int maxHistory = 32;    // Conversation context length
   }
   ```

   The key fields are: The `systemMessage`, This is your LLM prompt that defines who the NPC is. **The `greetingMessage` is the first thing the NPC says when they're activated. **The `failureMessage` is the fallback response if the AI can't generate a response. \*\*The `maxHistory` is the number of messages to keep in context for the AI to reference. The other key element is the `maxHistory` setting. This is where you trade off context quality vs. latency. Higher values mean the NPC remembers more of the conversation but takes longer to generate responses as the context window grows.

1. Initialize the manager  
   This manager handles the lifecycle—starting up the RTC engine, creating the AI agent, and managing the audio channel:

   ```csharp
   public class ConvoAIManager : MonoBehaviour
   {
       [SerializeField] private ConvoAIConfigs config;
       private JoinConvoAIChannelAudio convoAI;

       private void Start()
       {
            // Get the ConvoAI component
            convoAI = GetComponent<JoinConvoAIChannelAudio>();

            // Assign configuration
            convoAI.config = config;

            // Initialize the system
            InitializeConvoAI();
       }

       private void InitializeConvoAI()
       {
            // Configure audio settings for game context
            ConfigureForUnity();

            // Create the AI agent in the current channel
            convoAI.CreateConvoAIAgent();

            Debug.Log("ConvoAI system initialized successfully");
       }

       private void ConfigureForUnity()
       {
            // Set Unity-specific optimizations
            config.audioProfile = AudioProfile.GameOptimized;
            config.enableResponseCaching = true;
            config.maxConcurrentRequests = 1;
       }
   }
   ```

   What's happening when you call CreateConvoAIAgent():

   1. Agora's backend spins up an AI agent instance with your personality config
   1. That agent joins your RTC channel using agentRtcUid
   1. The agent starts listening for audio from the player
   1. Once it hears speech, ASR → LLM → TTS pipeline kicks off

   You'll get callbacks (next section) when the agent is ready and when it starts/stops speaking.

1. React to agent state changes
   NPCs aren't always available—they might be processing, joining the channel, or recovering from an error. Implement `IConvoAIStatusListener` to handle these states:

   ```csharp
   public interface IConvoAIStatusListener
   {
       void OnAgentStatusChanged(bool isActive, string agentId);
       void OnResponseReceived(string response, float confidence);
       void OnErrorOccurred(string error);
   }

   public class GameNPC : MonoBehaviour, IConvoAIStatusListener
   {
       [SerializeField] private string npcName = "Village Elder";
       [SerializeField] private Animator npcAnimator;

       public void OnAgentStatusChanged(bool isActive, string agentId)
       {
           if (isActive)
           {
               // NPC is ready for voice interaction
               EnableVoiceInteraction();
               ShowConversationUI();
               Debug.Log($"{npcName} is ready for conversation");
           }
           else
           {
               // NPC is processing or offline
               DisableVoiceInteraction();
               ShowWaitingIndicator();
           }
       }

       private void EnableVoiceInteraction()
       {
           // Update NPC animation state
           npcAnimator.SetBool("IsListening", true);

           // Show voice input indicator
           GameObject.Find("VoiceInputIndicator").SetActive(true);
       }

       private void DisableVoiceInteraction()
       {
           npcAnimator.SetBool("IsListening", false);
           GameObject.Find("VoiceInputIndicator").SetActive(false);
       }
   }
   ```

   Key thing to understand: When `OnResponseReceived()` fires, the TTS audio is already playing through Agora's audio channel. You don't need to manually trigger audio playback—just sync your animations and UI.

   > Common gotcha: If multiple NPCs use the same UID, they'll fight for the channel slot and keep dropping each other.

### Configure NPC personalities

Writing good NPC personalities for voice AI is different from text-based chatbots. Players won't wait through a 5-second response while your LLM generates three paragraphs of lore. You're optimizing for natural conversation cadence, not narrative depth.

### Design principles for voice NPCs

Before we write code, here's what actually works:  
Keep responses SHORT. Target 1-2 sentences (15-30 words). Anything longer and you're adding latency + players lose interest. The TTS needs time to generate audio, and long responses kill the flow.  
Be specific about speaking style. "Gruff" is vague. "Short sentences, no pleasantries, uses blacksmith metaphors" gives the LLM clear constraints.  
Limit knowledge domains. An NPC that "knows everything" becomes a help desk. Give them 2-3 specific expertise areas and explicitly tell the LLM to deflect other topics in character.  
Plan for context overflow. That maxHistory = 32 setting? Players will hit it. Decide now whether old conversation turns should drop off or if you'll summarize them.

1. Build a personality system
   Here's a structured approach to defining NPC personalities:

   ```csharp
   [System.Serializable]
   public class NPCPersonality
   {
       public string characterName;
       public string characterRole;    // "blacksmith", "quest giver", "merchant"
       public string speakingStyle;    // The "voice" - tone, speech patterns, quirks
       public string[] knowledgeAreas; // Specific topics they can discuss
       public string backgroundStory; // Minimal lore, not a novel

       public string GenerateSystemMessage()
       {
           // Build a prompt that emphasizes brevity and character consistency
           return $@"You are {characterName}, a {characterRole}.
                   Your speaking style is {speakingStyle}.
                   You have knowledge about: {string.Join(", ", knowledgeAreas)}.
                   Background: {backgroundStory}

                   Always stay in character and respond appropriately to the player's questions.
                   Keep responses concise but engaging, typically 1-2 sentences.";
       }
   }
   ```

   This generates a system message that's specific enough to get consistent behavior but flexible enough to handle unexpected player questions.

   NPCs need to load these personalities when they spawn and potentially update them based on game state:

   ```csharp
   public class PersonalityManager : MonoBehaviour
   {
       [SerializeField] private NPCPersonality[] availablePersonalities;
       [SerializeField] private int currentPersonalityIndex = 0;

       private JoinConvoAIChannelAudio convoAI;

       private void Start()
       {
           convoAI = GetComponent<JoinConvoAIChannelAudio>();
           ApplyPersonality(currentPersonalityIndex);
       }

       public void ApplyPersonality(int personalityIndex)
       {
           if (personalityIndex < 0 || personalityIndex >= availablePersonalities.Length)
               return;

           var personality = availablePersonalities[personalityIndex];

           // Update system message with personality
           convoAI.config.systemMessage = personality.GenerateSystemMessage();
           convoAI.config.agentName = personality.characterName;
           convoAI.config.greetingMessage = GenerateGreeting(personality);

           // Update the running agent with new personality
           convoAI.UpdateConvoAIAgent();

           Debug.Log($"Applied personality: {personality.characterName}");
       }

       private string GenerateGreeting(NPCPersonality personality)
       {
           // Generate contextual greeting based on personality
           switch (personality.characterRole.ToLower())
           {
               case "merchant":
                   return "Welcome, traveler! What goods interest you today?";
               case "guard":
                   return "Halt! State your business in this area.";
               case "scholar":
                   return "Greetings! I see you seek knowledge. How may I assist?";
               default:
                   return $"Hello there! I'm {personality.characterName}. How can I help?";
           }
       }
   }
   ```

   > Important: You can't update an agent's personality mid-conversation. The LLM's context is baked in when you create the agent. To change personality, you need to destroy and recreate the agent, which resets conversation history.

2. Inject game state into conversations

   The real magic happens when NPCs respond to your game world. To make NPCs truly interactive, you need to inject game state into conversations. This could be player level, quest progress, location, time of day, etc.

   Here's how to add game state to personalities to make them context-aware without overloading the system message:

   ```csharp
   public class ContextAwareNPC : MonoBehaviour
   {
       [SerializeField] private NPCPersonality basePersonality;
       private JoinConvoAIChannelAudio convoAI;
       private PlayerProgress playerProgress;

       private void Start()
       {
           convoAI = GetComponent<JoinConvoAIChannelAudio>();
           playerProgress = FindObjectOfType<PlayerProgress>();

           // Configure initial personality
           ConfigurePersonalityForContext();
       }

       private void ConfigurePersonalityForContext()
       {
           // Build context-aware system message
           var contextualMessage = BuildContextualSystemMessage();

           convoAI.config.systemMessage = contextualMessage;
           convoAI.config.greetingMessage = GenerateContextualGreeting();

           // Update the agent with new configuration
           convoAI.UpdateConvoAIAgent();
       }

       private string BuildContextualSystemMessage()
       {
           var baseMessage = basePersonality.GenerateSystemMessage();

           // Add game context
           var gameContext = $@"
                   Game Context:
                   - Player Level: {playerProgress.Level}
                   - Completed Quests: {playerProgress.CompletedQuests.Count}
                   - Current Location: {playerProgress.CurrentArea}
                   - Time of Day: {GetGameTimeOfDay()}

                   Adapt your responses based on this context.";

           return baseMessage + gameContext;
       }

       private string GenerateContextualGreeting()
       {
           // Generate greetings based on player progress
           if (playerProgress.Level < 5)
           {
               return "Ah, a newcomer! Welcome to our village.";
           }
           else if (playerProgress.CompletedQuests.Count > 10)
           {
               return "Well met, experienced adventurer! Your reputation precedes you.";
           }
           else
           {
               return "Good to see you again, traveler.";
           }
       }
   }
   ```

   Game developers should understand this is a critical tradeoff: Adding game context to the system message makes responses more relevant but increases token usage and latency. Find the minimum context needed—the LLM doesn't need to know every item in the player's inventory, just broad state like "player is wealthy" or "player has rare sword."

### Handle voice interactions

Voice interactions with NPCs introduce timing challenges you don't get with click-to-talk dialogue trees. Players expect conversations to feel responsive (low latency), natural (no awkward pauses), and context-aware (NPCs should react to proximity and game state).

Let's break down the interaction lifecycle and handle the tricky parts. Here's what happens from a player's perspective:

1. Player approaches NPC → Proximity detection
2. Player initiates conversation → Agent activation (ConvoAI starts listening)
3. Player speaks → ASR processes audio → LLM generates response → TTS plays audio
4. Player continues or walks away → Conversation ends, agent deactivates

Each stage has technical gotchas. Let's tackle them.

1.  Proximity-based activation
    The biggest mistake I see is creating all AI agents at scene start. That's expensive—each agent maintains an RTC connection and burns API credits even when idle. Better approach: activate agents only when players get close.

    Here's a proximity-based system that handles agent lifecycle::

    ```csharp
    public class VoiceInteractionHandler : MonoBehaviour, IConvoAIStatusListener
    {
        [SerializeField] private float interactionRange = 5f;
        [SerializeField] private LayerMask playerLayer;

        private JoinConvoAIChannelAudio convoAI;
        private Transform playerTransform;
        private bool isPlayerInRange = false;
        private bool isConversationActive = false;

        private void Start()
        {
            convoAI = GetComponent<JoinConvoAIChannelAudio>();
            playerTransform = GameObject.FindGameObjectWithTag("Player").transform;
        }

        private void Update()
        {
            CheckPlayerProximity();
            HandleInteractionState();
        }

        private void CheckPlayerProximity()
        {
            float distance = Vector3.Distance(transform.position, playerTransform.position);
            bool wasInRange = isPlayerInRange;
            isPlayerInRange = distance <= interactionRange;

            // Player entered interaction range
            if (isPlayerInRange && !wasInRange)
            {
                OnPlayerEnterRange();
            }
            // Player left interaction range
            else if (!isPlayerInRange && wasInRange)
            {
                OnPlayerExitRange();
            }
        }

        private void OnPlayerEnterRange()
        {
            Debug.Log("Player entered NPC interaction range");

            // Activate ConvoAI agent when player is nearby
            if (!convoAI.IsAgentActive())
            {
                convoAI.CreateConvoAIAgent();
            }

            // Show interaction prompt
            ShowInteractionPrompt(true);
        }

        private void OnPlayerExitRange()
        {
            Debug.Log("Player left NPC interaction range");

            // Deactivate agent to save resources
            if (convoAI.IsAgentActive() && !isConversationActive)
            {
                convoAI.StopConvoAIAgent();
            }

            // Hide interaction prompt
            ShowInteractionPrompt(false);
        }

        public void OnAgentStatusChanged(bool isActive, string agentId)
        {
            if (isActive)
            {
                Debug.Log($"NPC {agentId} is ready for conversation");
                EnableConversationMode();
            }
            else
            {
                Debug.Log($"NPC {agentId} conversation ended");
                DisableConversationMode();
            }
        }

        public void OnResponseReceived(string response, float confidence)
        {
            // Handle AI response with confidence scoring
            if (confidence > 0.7f)
            {
                DisplayResponse(response, ResponseType.Confident);
            }
            else
            {
                DisplayResponse(response, ResponseType.Uncertain);
            }

            // Track conversation metrics
            TrackConversationMetrics(response, confidence);
        }

        public void OnErrorOccurred(string error)
        {
            Debug.LogWarning($"ConvoAI Error: {error}");

            // Provide fallback response
            DisplayResponse(convoAI.config.failureMessage, ResponseType.Fallback);
        }
    }
    ```

    The agent creation happens in `OnPlayerEnterRange()`, giving you ~1-2 seconds of agent initialization time while the player approaches. By the time they're ready to talk, the agent is already active and listening. This hides the startup latency.

    The `!isConversationActive` check in `OnPlayerExitRange()` is critical—if a player walks away mid-conversation, you don't want to immediately kill the agent. Maybe they just took a step back. You'll handle conversation cleanup separately (next section).

    > Tradeoff to consider: This uses `Update()` for distance checks, which is fine for a handful of NPCs but doesn't scale to 50+ characters. For dense NPC scenes, use trigger colliders instead or spatial hashing.

2.  Conversation state management
    Voice conversations don't have clear boundaries like clicking "End Dialogue." Players might stop talking, walk away, or the network might drop. You need a state machine that handles timeouts and graceful exits:

    ```csharp
    public enum ConversationState
    {
        Idle,
        Listening,
        Processing,
        Responding,
        Ended
    }

    public class ConversationManager : MonoBehaviour
    {
        [SerializeField] private ConversationState currentState = ConversationState.Idle;
        [SerializeField] private float responseTimeout = 10f;
        [SerializeField] private int maxTurns = 10;

        private JoinConvoAIChannelAudio convoAI;
        private int currentTurnCount = 0;
        private Coroutine timeoutCoroutine;

        private void Start()
        {
            convoAI = GetComponent<JoinConvoAIChannelAudio>();
        }

        public void StartConversation()
        {
            if (currentState != ConversationState.Idle) return;

            currentState = ConversationState.Listening;
            currentTurnCount = 0;

            // Create ConvoAI agent for conversation
            convoAI.CreateConvoAIAgent();

            // Start response timeout monitoring
            StartResponseTimeout();

            Debug.Log("Conversation started");
        }

        public void EndConversation()
        {
            currentState = ConversationState.Ended;

            // Stop ConvoAI agent
            convoAI.StopConvoAIAgent();

            // Cancel any pending timeouts
            if (timeoutCoroutine != null)
            {
                StopCoroutine(timeoutCoroutine);
                timeoutCoroutine = null;
            }

            Debug.Log("Conversation ended");
        }

        private void StartResponseTimeout()
        {
            if (timeoutCoroutine != null)
                StopCoroutine(timeoutCoroutine);

            timeoutCoroutine = StartCoroutine(ResponseTimeoutCoroutine());
        }

        private IEnumerator ResponseTimeoutCoroutine()
        {
            yield return new WaitForSeconds(responseTimeout);

            // Handle conversation timeout
            if (currentState == ConversationState.Processing)
            {
                Debug.LogWarning("Conversation timed out");
                OnResponseReceived(convoAI.config.failureMessage, 0f);
            }
        }

        private void OnResponseReceived(string response, float confidence)
        {
            currentState = ConversationState.Responding;
            currentTurnCount++;

            // Cancel timeout
            if (timeoutCoroutine != null)
            {
                StopCoroutine(timeoutCoroutine);
                timeoutCoroutine = null;
            }

            // Display response to player
            DisplayNPCResponse(response);

            // Check if conversation should continue
            if (currentTurnCount >= maxTurns)
            {
                EndConversation();
            }
            else
            {
                // Return to listening state
                currentState = ConversationState.Listening;
                StartResponseTimeout();
            }
        }

        private void DisplayNPCResponse(string response)
        {
            // Display response in UI
            var dialogueUI = FindObjectOfType<DialogueUI>();
            if (dialogueUI != null)
            {
                dialogueUI.ShowNPCMessage(response);
            }

            // Update NPC animation
            var animator = GetComponent<Animator>();
            if (animator != null)
            {
                animator.SetTrigger("Speak");
            }
        }
    }
    ```

    The responseTimeout of 10 seconds handles the worst-case scenario: LLM hangs, network drops, or ASR can't transcribe the audio. Without this, players sit waiting indefinitely while your NPC stares blankly. The timeout triggers a fallback response from config.failureMessage so the conversation doesn't just... stop.
    The maxTurns limit prevents infinite conversations. Even with good personalities, LLMs can get stuck in loops. After 10 turns, force an exit. You can tune this based on your game—quest-giver NPCs might need 3-5 turns, while companion characters could go 20+.
    State transitions:

    - `Idle` → `Listening`: Player initiates conversation
    - `Listening` → `Processing`: ASR detects speech and sends to LLM
    - `Processing` → `Responding`: LLM returns text, TTS starts playing
    - `Responding` → `Listening`: TTS finishes, ready for next turn
    - Any state → `Ended`: Player leaves, maxTurns hit, or manual exit

    The currentState tracking is essential for debugging. When something breaks (and it will), knowing whether you're stuck in Processing vs. Responding tells you if it's an LLM issue or audio playback issue.

3.  Spatial audio integration

    If your game is first-person or third-person with 3D audio, you want NPC voices to feel positional. This requires routing Agora's audio streams through Unity's AudioSource system with spatial blend:

    ```csharp
    public class AdvancedNPCInteraction : MonoBehaviour
    {
        [SerializeField] private AudioSource npcAudioSource;
        [SerializeField] private float spatialAudioRange = 10f;

        private JoinConvoAIChannelAudio convoAI;
        private Transform playerTransform;

        private void Start()
        {
            convoAI = GetComponent<JoinConvoAIChannelAudio>();
            playerTransform = GameObject.FindGameObjectWithTag("Player").transform;

            ConfigureSpatialAudio();
        }

        private void ConfigureSpatialAudio()
        {
            // Set up 3D spatial audio for immersive conversations
            npcAudioSource.spatialBlend = 1f; // Full 3D audio
            npcAudioSource.rolloffMode = AudioRolloffMode.Logarithmic;
            npcAudioSource.maxDistance = spatialAudioRange;
            npcAudioSource.minDistance = 1f;
        }

        private void Update()
        {
            UpdateSpatialAudio();
        }

        private void UpdateSpatialAudio()
        {
            // Adjust audio volume based on player distance
            float distance = Vector3.Distance(transform.position, playerTransform.position);
            float volume = Mathf.Clamp01(1f - (distance / spatialAudioRange));

            npcAudioSource.volume = volume;

            // Update ConvoAI audio settings based on proximity
            if (distance <= spatialAudioRange * 0.5f)
            {
                // Player is close - use high quality audio
                convoAI.config.audioProfile = AudioProfile.HighQuality;
            }
            else
            {
                // Player is far - use standard quality to save bandwidth
                convoAI.config.audioProfile = AudioProfile.Standard;
            }
        }

        public void HandleInteractionInput()
        {
            // Handle player interaction input (key press, button click, etc.)
            if (Input.GetKeyDown(KeyCode.E))
            {
                ToggleConversation();
            }
        }

        private void ToggleConversation()
        {
            if (convoAI.IsAgentActive())
            {
                // End conversation
                convoAI.StopConvoAIAgent();
                Debug.Log("Conversation ended by player");
            }
            else
            {
                // Start conversation
                convoAI.CreateConvoAIAgent();
                Debug.Log("Conversation started by player");
            }
        }
    }
    ```

    The spatialBlend = 1f makes the audio fully 3D-positioned. If you're building a top-down or 2D game, set this to 0f for pure stereo audio instead.

    The distance-based audio quality switching (AudioProfile.HighQuality vs. AudioProfile.Standard) is a bandwidth optimization. Close conversations get better TTS quality, distant NPCs use compressed audio. Players won't notice the quality difference at range, but your network usage will drop.
    Manual toggle pattern: The HandleInteractionInput() method shows an alternative to auto-activation. Some games want explicit "press E to talk" interactions rather than automatic proximity detection. Use this if:

    - You have dense NPC crowds (don't want to trigger all of them)
    - You need UI-driven conversation initiation (click on NPC portrait)
    - You're building a stealth game (can't have NPCs auto-activating when you're hiding nearby)

## Test

Testing voice AI NPCs is harder than testing traditional dialogue systems because there's no deterministic output. The same player input can generate different LLM responses, network conditions vary, and ASR accuracy depends on microphone quality. You're not just checking "does it work"—you're validating behavior across a range of failure modes.

To ensure that you have implemented conversational AI NPCs correctly:

1. Test basic functionality in Unity Play Mode:

   - Start the game and approach an NPC within interaction range
   - Verify that the ConvoAI agent activates (check console logs)
   - Speak to the NPC and confirm you receive appropriate responses

2. Test NPC personality consistency:

   - Have extended conversations with different NPCs
   - Verify each NPC maintains their configured personality traits
   - Check that responses are contextually appropriate to the game world

3. Test performance and resource management:

   - Monitor memory usage during conversations
   - Verify that agents properly deactivate when players leave interaction range
   - Test multiple concurrent NPC conversations if your game supports it

4. Test error handling:
   - Simulate network interruptions during conversations
   - Verify fallback responses work when AI services are unavailable
   - Test conversation timeouts and recovery

## Next steps

You've successfully implemented intelligent, conversational AI NPCs in Unity using Agora's ConvoAI Engine! You now have NPCs that can engage in natural, dynamic conversations with players while maintaining distinct personalities and game context awareness. The complete source code for this tutorial is available on [GitHub](https://github.com/AgoraIO-Community/AgoraConvoAI_Unity).

Your next steps after implementing this functionality:

- Explore [Advanced ConvoAI Features](https://docs.agora.io/en/conversational-ai/) to enhance your NPCs with multi-language support and advanced AI capabilities
- Join the [Agora.io Developer Community](https://agora-community.slack.com/) to connect with other developers building conversational AI experiences

## Reference

For more information about Agora's ConvoAI Engine, see the [Agora ConvoAI Engine documentation](https://docs.agora.io/en/conversational-ai/). Check out the [full code for this tutorial](https://github.com/chitimalli/AgoraConvoAI_Unity) on GitHub.
