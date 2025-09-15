# Build Intelligent NPCs with Agora's ConvoAI Engine in Unity

Real-time conversational AI transforms game experiences by enabling NPCs that understand, respond, and adapt to player voice input in natural, dynamic ways. With Agora's ConvoAI Engine, you can easily implement intelligent NPCs that create truly immersive gaming experiences while leveraging the same enterprise-grade technology powering Fortune 500 communications platforms.

This page shows you how to add intelligent, voice-enabled NPCs to your Unity game using Agora's ConvoAI Engine and the comprehensive Unity integration package.

## Understand the tech

To implement conversational AI NPCs with Agora, you use the Agora ConvoAI Engine that works in the following way:

1. Your Unity game initializes the Agora RTC client
2. NPCs join the same audio/video channels as players using unique agent IDs
3. The ConvoAI Engine processes player voice input through ASR (speech-to-text)
4. LLM generates contextual responses based on NPC personality configuration
5. TTS converts AI responses to natural speech delivered through Agora's network
6. The SDK manages all real-time communication complexities including network optimization and quality adaptation

The following figure shows the data flow between your Unity application, players, and Agora's ConvoAI infrastructure:

![ConvoAI Architecture]
<img width="1222" height="682" alt="ConvoAI_Architecture" src="https://github.com/user-attachments/assets/b1ed62ed-bd33-4e54-bfde-4d915fec0eb9" />


The ConvoAI Engine seamlessly integrates with Agora's proven Voice SDK, allowing NPCs to participate naturally in player conversations while maintaining ultra-low latency and enterprise-grade reliability.

## Prerequisites

To successfully implement conversational AI NPCs with Agora, you must have the following:

• A valid Agora account. If you don't have an account, see how to [Get started with Agora](https://docs.agora.io/en/Agora%20Platform/get_appid_token?platform=All%20Platforms&utm_source=medium&utm_medium=blog&utm_campaign=Build_Intelligent_NPCs_with_Agora_ConvoAI_Engine_in_Unity).

• An Agora project with ConvoAI Engine access enabled.

• Unity 2020.3 or later with the Agora RTC SDK for Unity installed.

• Basic knowledge of C# and Unity development.

• API credentials for your chosen LLM provider (OpenAI, Azure OpenAI, etc.).

• TTS service credentials (Azure Cognitive Services, Amazon Polly, etc.).

## Project setup

To create the environment necessary to build intelligent NPCs with Agora's ConvoAI Engine:

1. Clone the [AgoraConvoAI_Unity repository](https://github.com/chitimalli/AgoraConvoAI_Unity) into your Unity project.

2. Import the Agora RTC SDK for Unity from the `Assets/Agora-RTC-Plugin/` directory.

3. Create a ConvoAI configuration asset in your project:
   - Right-click in Project window → Create → Agora → ConvoAIConfigs
   - Name it `ConvoAIConfigs`

4. Set up your Unity scene with the following structure:

```
Scene Hierarchy:
├── ConvoAI Manager (Empty GameObject)
│   └── JoinConvoAIChannelAudio (Component)
├── NPC Character (3D Model)
│   └── GameNPC (Custom Script)
└── UI Canvas
    ├── Voice Input Indicator
    └── NPC Status Display
```

## Build

Implementing conversational AI NPCs with Agora involves configuring the ConvoAI system and creating intelligent NPC behaviors. This section shows you how to:

• [Initialize the ConvoAI system and create AI agents](#initialize-convoai-system)
• [Configure NPC personalities and behaviors](#configure-npc-personalities)
• [Handle voice interactions and responses](#handle-voice-interactions)

### Initialize ConvoAI system

To set up the basic ConvoAI system in your Unity game:

1. First, configure your ConvoAI settings:

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

2. Next, implement the basic ConvoAI manager:

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
        // Configure for Unity game environment
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

3. Implement the interface for NPC status listening:

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

### Configure NPC personalities

To create distinct and engaging NPC personalities using the ConvoAI system:

1. First, implement a personality configuration system:

```csharp
[System.Serializable]
public class NPCPersonality
{
    public string characterName;
    public string characterRole;
    public string speakingStyle;
    public string[] knowledgeAreas;
    public string backgroundStory;
    
    public string GenerateSystemMessage()
    {
        return $@"You are {characterName}, a {characterRole}. 
                 Your speaking style is {speakingStyle}.
                 You have knowledge about: {string.Join(", ", knowledgeAreas)}.
                 Background: {backgroundStory}
                 
                 Always stay in character and respond appropriately to the player's questions.
                 Keep responses concise but engaging, typically 1-2 sentences.";
    }
}

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

2. Next, implement dynamic personality updates based on game state:

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

### Handle voice interactions

To implement responsive voice interaction handling in your NPCs:

1. First, implement voice input detection and processing:

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

2. Next, implement conversation state management:

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

3. Finally, implement advanced interaction features:

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

## Test

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

You've successfully implemented intelligent, conversational AI NPCs in Unity using Agora's ConvoAI Engine! You now have NPCs that can engage in natural, dynamic conversations with players while maintaining distinct personalities and game context awareness. The complete source code for this tutorial is available on [GitHub](https://github.com/chitimalli/AgoraConvoAI_Unity).

Your next steps after implementing this functionality:

• Explore [Advanced ConvoAI Features](https://docs.agora.io/en/conversational-ai/) to enhance your NPCs with multi-language support and advanced AI capabilities
• Join the [Agora.io Developer Community](https://agora-community.slack.com/) to connect with other developers building conversational AI experiences

## Reference

For more information about Agora's ConvoAI Engine, see the [Agora ConvoAI Engine documentation](https://docs.agora.io/en/conversational-ai/). Check out the [full code for this tutorial](https://github.com/chitimalli/AgoraConvoAI_Unity) on GitHub.
