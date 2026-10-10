# Tech Library Development Blueprint

This blueprint provides a complete, spoon-fed guide for developing the Tech Library educational simulation game from scratch, including GitHub repository setup, development environment configuration, project structure, and the full development lifecycle.

## Technology Stack

**Frontend & Backend**: Blazor WebAssembly (client-side C#)
- Meets C# backend requirement
- Compiles to WebAssembly, runs entirely in browser
- Can be deployed to Vercel as static files
- Seamless asset integration
- No separate backend server needed

**AI Integration**: OpenAI API
- Easiest to integrate with comprehensive documentation
- Reliable and credible
- Generous free tier for development
- Excellent SDK for .NET

**Data Storage**: Browser Storage (localStorage/IndexedDB)
- No database required
- Persistent game states
- Fast and simple

**Deployment**: Vercel
- Free hosting for static sites
- Automatic deployments from GitHub
- Global CDN

**Development Environment**: VS Code with C# Dev Kit extension

## Phase 1: GitHub Repository Setup

### Step 1.1: Create GitHub Repository
1. Go to https://github.com/new
2. Repository name: `tech-library-game`
3. Description: "First-person educational simulation game where players act as IT librarians"
4. Set to **Public** or **Private** (your preference)
5. Initialize with:
   - [x] Add a README file
   - [x] Add .gitignore (select .NET)
   - [ ] Choose a license (optional)
6. Click "Create repository"

### Step 1.2: Clone Repository Locally
1. Open VS Code
2. Open terminal (Ctrl + `)
3. Navigate to your desired workspace directory:
   ```powershell
   cd C:\Users\Admin944\Projects
   ```
4. Clone the repository:
   ```powershell
   git clone https://github.com/YOUR_USERNAME/tech-library-game.git
   cd tech-library-game
   ```

### Step 1.3: Configure Git
```powershell
git config user.name "Your Name"
git config user.email "your.email@example.com"
```

## Phase 2: Development Environment Setup

### Step 2.1: Install .NET SDK
1. Download .NET 8.0 SDK from https://dotnet.microsoft.com/download
2. Run the installer (Windows)
3. Verify installation:
   ```powershell
   dotnet --version
   ```
   Should output: 8.0.xxx or higher

### Step 2.2: Install VS Code Extensions
Open VS Code and install these extensions:
1. **C# Dev Kit** (Microsoft)
2. **Blazor WebAssembly IntelliSense** (optional but recommended)
3. **GitLens** (for better Git integration)
4. **Prettier** (code formatting)

### Step 2.3: Install Node.js (for Vercel CLI)
1. Download from https://nodejs.org/
2. Install LTS version
3. Verify installation:
   ```powershell
   node --version
   npm --version
   ```

### Step 2.4: Install Vercel CLI
```powershell
npm install -g vercel
```

## Phase 3: Project Initialization

### Step 3.1: Create Blazor WebAssembly Project
In the project directory:
```powershell
dotnet new blazorwasm -n TechLibrary -o TechLibrary
cd TechLibrary
```

### Step 3.2: Project Structure Overview
The created structure will be:
```
TechLibrary/
├── Client/               # Blazor WebAssembly client
│   ├── Pages/           # Razor pages
│   ├── Components/      # Reusable components
│   ├── wwwroot/         # Static assets (CSS, JS, images)
│   └── Program.cs       # Entry point
├── Server/              # (Not needed for our approach)
└── Shared/              # (Not needed for our approach)
```

Since we're using pure client-side Blazor, we'll work primarily in the `Client` folder.

### Step 3.3: Clean Up Unnecessary Files
```powershell
# Remove Server and Shared folders (not needed)
cd Client
```

### Step 3.4: Install Required NuGet Packages
```powershell
dotnet add package Microsoft.Extensions.Http
dotnet add package OpenAI
dotnet add package Blazored.LocalStorage
```

## Phase 4: Project Configuration

### Step 4.1: Configure Program.cs
Open `Client/Program.cs` and replace with:

```csharp
using Microsoft.AspNetCore.Components.Web;
using Microsoft.AspNetCore.Components.WebAssembly.Hosting;
using TechLibrary;
using Blazored.LocalStorage;

var builder = WebAssemblyHostBuilder.CreateDefault(args);
builder.RootComponents.Add<App>("#app");
builder.RootComponents.Add<HeadOutlet>("head::after");

// Add HttpClient for API calls
builder.Services.AddScoped(sp => new HttpClient { BaseAddress = new Uri(builder.HostEnvironment.BaseAddress) });

// Add Local Storage service
builder.Services.AddBlazoredLocalStorage();

// Add OpenAI service (API key will be set later)
builder.Services.AddScoped<OpenAIService>();

await builder.Build().RunAsync();
```

### Step 4.2: Create Services Folder Structure
```powershell
mkdir Services
mkdir Models
mkdir Data
```

### Step 4.3: Create OpenAI Configuration
Create `Client/appsettings.json`:
```json
{
  "OpenAI": {
    "ApiKey": "YOUR_OPENAI_API_KEY_HERE",
    "Model": "gpt-3.5-turbo"
  }
}
```

**Important**: Never commit the actual API key. We'll handle this with environment variables.

### Step 4.4: Update .gitignore
Add to `.gitignore`:
```
appsettings.json
appsettings.*.json
!.gitignore
```

## Phase 5: Core Game Architecture

### Step 5.1: Create Data Models

**Create `Client/Models/Book.cs`:**
```csharp
namespace TechLibrary.Models;

public class Book
{
    public int Id { get; set; }
    public string Title { get; set; }
    public string Category { get; set; }
    public string Summary { get; set; }
    public string Content { get; set; }
    public string Difficulty { get; set; }
}
```

**Create `Client/Models/CustomerRequest.cs`:**
```csharp
namespace TechLibrary.Models;

public class CustomerRequest
{
    public string Topic { get; set; }
    public string Category { get; set; }
    public string Question { get; set; }
}
```

**Create `Client/Models/GameState.cs`:**
```csharp
namespace TechLibrary.Models;

public class GameState
{
    public int CurrentLevel { get; set; } = 1;
    public int CustomersServed { get; set; } = 0;
    public int Score { get; set; } = 0;
    public DateTime LastPlayed { get; set; } = DateTime.Now;
    public List<string> CompletedTopics { get; set; } = new();
}
```

### Step 5.2: Create OpenAI Service

**Create `Client/Services/OpenAIService.cs`:**
```csharp
using OpenAI;
using OpenAI.Chat;
using TechLibrary.Models;

namespace TechLibrary.Services;

public class OpenAIService
{
    private readonly string _apiKey;
    private readonly string _model;

    public OpenAIService(IConfiguration configuration)
    {
        _apiKey = configuration["OpenAI:ApiKey"] ?? Environment.GetEnvironmentVariable("OPENAI_API_KEY");
        _model = configuration["OpenAI:Model"] ?? "gpt-3.5-turbo";
    }

    public async Task<string> GenerateBookContent(string topic, string category)
    {
        var client = new OpenAIClient(_apiKey);
        var chatClient = client.GetChatClient(_model);

        var messages = new List<ChatMessage>
        {
            new SystemChatMessage("You are an IT educational content generator. Generate concise, accurate summaries about IT topics."),
            new UserChatMessage($"Generate a brief summary about {topic} in the category of {category}. Include key concepts and keep it under 200 words.")
        };

        var response = await chatClient.CompleteChatAsync(messages);
        return response.Value.Content[0].Text;
    }

    public async Task<string> GenerateFollowUpQuestion(string topic, string summary)
    {
        var client = new OpenAIClient(_apiKey);
        var chatClient = client.GetChatClient(_model);

        var messages = new List<ChatMessage>
        {
            new SystemChatMessage("You are an IT quiz generator. Create simple comprehension questions based on provided content."),
            new UserChatMessage($"Based on this summary about {topic}: {summary}, generate one simple multiple-choice question to test understanding.")
        };

        var response = await chatClient.CompleteChatAsync(messages);
        return response.Value.Content[0].Text;
    }
}
```

### Step 5.3: Create Game State Service

**Create `Client/Services/GameStateService.cs`:**
```csharp
using Blazored.LocalStorage;
using TechLibrary.Models;

namespace TechLibrary.Services;

public class GameStateService
{
    private readonly ILocalStorageService _localStorage;
    private const string StorageKey = "techLibraryGameState";

    public GameStateService(ILocalStorageService localStorage)
    {
        _localStorage = localStorage;
    }

    public async Task<GameState> LoadGameStateAsync()
    {
        var exists = await _localStorage.ContainKeyAsync(StorageKey);
        if (exists)
        {
            return await _localStorage.GetItemAsync<GameState>(StorageKey);
        }
        return new GameState();
    public async Task SaveGameStateAsync(GameState state)
    {
        state.LastPlayed = DateTime.Now;
        await _localStorage.SetItemAsync(StorageKey, state);
    }

    public async Task ResetGameStateAsync()
    {
        await _localStorage.RemoveItemAsync(StorageKey);
    }
}
```

### Step 5.4: Create Book Data Service

**Create `Client/Services/BookDataService.cs`:**
```csharp
using TechLibrary.Models;

namespace TechLibrary.Services;

public class BookDataService
{
    private readonly List<Book> _initialBooks = new()
    {
        new Book { Id = 1, Title = "Object-Oriented Programming", Category = "Programming", Summary = "OOP is a programming paradigm based on objects containing data and code.", Difficulty = "Basic" },
        new Book { Id = 2, Title = "TCP/IP Protocol", Category = "Networking", Summary = "TCP/IP is the foundational communication protocol suite for the internet.", Difficulty = "Intermediate" },
        new Book { Id = 3, Title = "Computer Hardware Basics", Category = "Basic IT", Summary = "Computer hardware includes physical components like CPU, RAM, and storage devices.", Difficulty = "Basic" },
        new Book { Id = 4, Title = "Python Programming", Category = "IT Languages", Summary = "Python is a high-level, interpreted programming language known for readability.", Difficulty = "Basic" },
        new Book { Id = 5, Title = "Operating Systems", Category = "Computer Systems", Summary = "OS manages computer hardware and software resources and provides services.", Difficulty = "Intermediate" }
    };

    public List<Book> GetInitialBooks() => _initialBooks;

    public List<Book> FilterByCategory(List<Book> books, string category)
    {
        return books.Where(b => b.Category == category).ToList();
    }

    public Book? FindByTitle(List<Book> books, string title)
    {
        return books.FirstOrDefault(b => b.Title.Equals(title, StringComparison.OrdinalIgnoreCase));
    }
}
```

## Phase 6: UI Component Development

### Step 6.1: Create Main Game Pages

**Create `Client/Pages/BootScreen.razor`:**
```razor
@page "/"

<div class="boot-screen">
    <h1>Tech Library</h1>
    <p class="subtitle">IT Librarian Simulation</p>
    
    <div class="menu-buttons">
        <button class="menu-btn" @onclick="StartGame">Start</button>
        <button class="menu-btn" @onclick="ShowSettings">Settings</button>
        <button class="menu-btn" @onclick="ShowAbout">About</button>
    </div>
</div>

@code {
    private void StartGame()
    {
        NavigationManager.NavigateTo("/game");
    }

    private void ShowSettings()
    {
        NavigationManager.NavigateTo("/settings");
    }

    private void ShowAbout()
    {
        NavigationManager.NavigateTo("/about");
    }
}
```

**Create `Client/Pages/Game.razor`:**
```razor
@page "/game"
@inject GameStateService GameStateService
@inject BookDataService BookDataService
@inject OpenAIService OpenAIService

<div class="game-container">
    @if (isLoading)
    {
        <div class="loading">Loading...</div>
    }
    else
    {
        <div class="library-desk">
            <h2>Library Desk</h2>
            <p class="customer-request">@currentRequest?.Topic</p>
            
            <div class="book-shelf">
                @foreach (var book in availableBooks)
                {
                    <div class="book-card" @onclick="() => SelectBook(book)">
                        <h3>@book.Title</h3>
                        <p>@book.Category</p>
                    </div>
                }
            </div>
            
            @if (selectedBook != null)
            {
                <div class="book-details">
                    <h3>@selectedBook.Title</h3>
                    <p>@selectedBook.Summary</p>
                    <button @onclick="GiveBookToCustomer">Give to Customer</button>
                </div>
            }
            
            <div class="game-controls">
                <button @onclick="PauseGame">Pause / Leave</button>
            </div>
        </div>
    }
</div>

@code {
    private GameState gameState = new();
    private CustomerRequest? currentRequest;
    private List<Book> availableBooks = new();
    private Book? selectedBook;
    private bool isLoading = true;

    protected override async Task OnInitializedAsync()
    {
        gameState = await GameStateService.LoadGameStateAsync();
        availableBooks = BookDataService.GetInitialBooks();
        GenerateNewRequest();
        isLoading = false;
    }

    private void GenerateNewRequest()
    {
        var randomBook = availableBooks[new Random().Next(availableBooks.Count)];
        currentRequest = new CustomerRequest
        {
            Topic = randomBook.Title,
            Category = randomBook.Category
        };
    }

    private void SelectBook(Book book)
    {
        selectedBook = book;
    }

    private async Task GiveBookToCustomer()
    {
        if (selectedBook != null)
        {
            gameState.CustomersServed++;
            await GameStateService.SaveGameStateAsync(gameState);
            NavigationManager.NavigateTo($"/educational/{selectedBook.Id}");
        }
    }

    private void PauseGame()
    {
        NavigationManager.NavigateTo("/");
    }
}
```

**Create `Client/Pages/EducationalPopup.razor`:**
```razor
@page "/educational/{bookId:int}"
@inject BookDataService BookDataService
@inject NavigationManager NavigationManager

<div class="educational-popup">
    <h2>@book?.Title</h2>
    <div class="content">
        <p>@book?.Summary</p>
        <div class="detailed-info">
            <h3>Key Concepts:</h3>
            <p>@educationalContent</p>
        </div>
    </div>
    <button @onclick="ContinueGame">Continue</button>
</div>

@code {
    [Parameter]
    public int BookId { get; set; }
    
    private Book? book;
    private string educationalContent = "Loading educational content...";

    protected override void OnInitialized()
    {
        book = BookDataService.GetInitialBooks().FirstOrDefault(b => b.Id == BookId);
        educationalContent = GenerateEducationalContent(book);
    }

    private string GenerateEducationalContent(Book? book)
    {
        if (book == null) return "Book not found.";
        
        return book.Category switch
        {
            "Programming" => "Programming involves writing instructions for computers to execute. Key concepts include variables, loops, functions, and data structures.",
            "Networking" => "Networking connects computers to share resources. Key concepts include IP addresses, protocols, routers, and security.",
            "Basic IT" => "Basic IT covers fundamental computer skills including hardware, software, and digital literacy.",
            "IT Languages" => "Programming languages are tools for creating software. Each has unique syntax and use cases.",
            "Computer Systems" => "Computer systems manage hardware and software resources to provide services to users and applications.",
            _ => "This topic covers important IT concepts relevant to modern technology."
        };
    }

    private void ContinueGame()
    {
        NavigationManager.NavigateTo("/game");
    }
}
```

**Create `Client/Pages/Settings.razor`:**
```razor
@page "/settings"

<div class="settings-page">
    <h2>Settings</h2>
    
    <div class="setting-item">
        <label>Music Volume</label>
        <input type="range" min="0" max="100" value="50" />
    </div>
    
    <div class="setting-item">
        <label>Sound Effects Volume</label>
        <input type="range" min="0" max="100" value="50" />
    </div>
    
    <div class="setting-item">
        <label>Text Size</label>
        <select>
            <option>Small</option>
            <option selected>Medium</option>
            <option>Large</option>
        </select>
    </div>
    
    <button @onclick="GoBack">Back</button>
</div>

@code {
    private void GoBack()
    {
        NavigationManager.NavigateTo("/");
    }
}
```

**Create `Client/Pages/About.razor`:**
```razor
@page "/about"

<div class="about-page">
    <h2>About Tech Library</h2>
    <p>Version 1.0.0</p>
    
    <div class="about-content">
        <p>Tech Library is an educational simulation game where you play as an IT librarian.</p>
        <p>Help customers find the right IT resources and learn along the way!</p>
        
        <h3>Features:</h3>
        <ul>
            <li>Learn IT concepts through gameplay</li>
            <li>AI-generated content for infinite replayability</li>
            <li>Progressive difficulty system</li>
            <li>Persistent save states</li>
        </ul>
        
        <h3>Credits:</h3>
        <p>Developed with Blazor WebAssembly and OpenAI</p>
    </div>
    
    <button @onclick="GoBack">Back</button>
</div>

@code {
    private void GoBack()
    {
        NavigationManager.NavigateTo("/");
    }
}
```

### Step 6.2: Update App.razor
Replace `Client/App.razor` with:
```razor
<Router AppAssembly="@typeof(App).Assembly">
    <Found Context="routeData">
        <RouteView RouteData="@routeData" DefaultLayout="@typeof(MainLayout)" />
        <FocusOnNavigate RouteData="@routeData" Selector="h1" />
    </Found>
    <NotFound>
        <PageTitle>Not found</PageTitle>
        <LayoutView Layout="@typeof(MainLayout)">
            <p role="alert">Sorry, there's nothing at this address.</p>
        </LayoutView>
    </NotFound>
</Router>
```

### Step 6.3: Update MainLayout.razor
Replace `Client/Shared/MainLayout.razor` with:
```razor
@inherits LayoutComponentBase

<div class="main-layout">
    @Body
</div>
```

## Phase 7: Styling

### Step 7.1: Update Main CSS
Replace `Client/wwwroot/css/app.css` with:
```css
* {
    margin: 0;
    padding: 0;
    box-sizing: border-box;
}

body {
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    min-height: 100vh;
    color: #333;
}

.main-layout {
    min-height: 100vh;
}

/* Boot Screen */
.boot-screen {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    min-height: 100vh;
    text-align: center;
    color: white;
}

.boot-screen h1 {
    font-size: 4rem;
    margin-bottom: 0.5rem;
}

.subtitle {
    font-size: 1.5rem;
    margin-bottom: 3rem;
    opacity: 0.9;
}

.menu-buttons {
    display: flex;
    flex-direction: column;
    gap: 1rem;
}

.menu-btn {
    padding: 1rem 3rem;
    font-size: 1.2rem;
    border: none;
    border-radius: 8px;
    background: white;
    color: #667eea;
    cursor: pointer;
    transition: transform 0.2s, box-shadow 0.2s;
}

.menu-btn:hover {
    transform: translateY(-2px);
    box-shadow: 0 4px 12px rgba(0,0,0,0.2);
}

/* Game Container */
.game-container {
    padding: 2rem;
    max-width: 1200px;
    margin: 0 auto;
}

.library-desk {
    background: white;
    border-radius: 16px;
    padding: 2rem;
    box-shadow: 0 8px 32px rgba(0,0,0,0.1);
}

.library-desk h2 {
    color: #667eea;
    margin-bottom: 1.5rem;
}

.customer-request {
    font-size: 1.3rem;
    background: #f0f4ff;
    padding: 1rem;
    border-radius: 8px;
    margin-bottom: 2rem;
    border-left: 4px solid #667eea;
}

.book-shelf {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
    gap: 1rem;
    margin-bottom: 2rem;
}

.book-card {
    background: #f8f9fa;
    padding: 1rem;
    border-radius: 8px;
    cursor: pointer;
    transition: transform 0.2s, background 0.2s;
    border: 2px solid transparent;
}

.book-card:hover {
    transform: translateY(-4px);
    background: #e9ecef;
    border-color: #667eea;
}

.book-card h3 {
    font-size: 1rem;
    margin-bottom: 0.5rem;
    color: #333;
}

.book-card p {
    font-size: 0.9rem;
    color: #666;
}

.book-details {
    background: #e8f4f8;
    padding: 1.5rem;
    border-radius: 8px;
    margin-bottom: 1rem;
}

.game-controls {
    display: flex;
    gap: 1rem;
}

.game-controls button {
    padding: 0.75rem 1.5rem;
    border: none;
    border-radius: 6px;
    background: #667eea;
    color: white;
    cursor: pointer;
    font-size: 1rem;
}

.game-controls button:hover {
    background: #5568d3;
}

/* Educational Popup */
.educational-popup {
    position: fixed;
    top: 0;
    left: 0;
    right: 0;
    bottom: 0;
    background: rgba(0,0,0,0.8);
    display: flex;
    align-items: center;
    justify-content: center;
    z-index: 1000;
}

.educational-popup > div {
    background: white;
    padding: 2rem;
    border-radius: 16px;
    max-width: 600px;
    max-height: 80vh;
    overflow-y: auto;
}

.educational-popup h2 {
    color: #667eea;
    margin-bottom: 1rem;
}

.detailed-info {
    margin-top: 1.5rem;
    padding-top: 1.5rem;
    border-top: 1px solid #eee;
}

.educational-popup button {
    margin-top: 1.5rem;
    padding: 0.75rem 2rem;
    border: none;
    border-radius: 6px;
    background: #667eea;
    color: white;
    cursor: pointer;
    font-size: 1rem;
}

/* Settings & About Pages */
.settings-page, .about-page {
    max-width: 800px;
    margin: 2rem auto;
    padding: 2rem;
    background: white;
    border-radius: 16px;
    box-shadow: 0 8px 32px rgba(0,0,0,0.1);
}

.setting-item {
    margin-bottom: 1.5rem;
}

.setting-item label {
    display: block;
    margin-bottom: 0.5rem;
    font-weight: 600;
}

.setting-item input, .setting-item select {
    width: 100%;
    padding: 0.75rem;
    border: 1px solid #ddd;
    border-radius: 6px;
}

.about-content {
    margin-top: 1.5rem;
    line-height: 1.6;
}

.about-content h3 {
    margin-top: 1.5rem;
    margin-bottom: 0.75rem;
    color: #667eea;
}

.about-content ul {
    margin-left: 1.5rem;
    margin-bottom: 1rem;
}

.about-content li {
    margin-bottom: 0.5rem;
}

.loading {
    text-align: center;
    font-size: 1.5rem;
    color: white;
    padding: 2rem;
}
```

## Phase 8: Testing and Debugging

### Step 8.1: Run Development Server
```powershell
dotnet run
```
The app will be available at https://localhost:5001

### Step 8.2: Test Core Functionality
1. **Boot Screen**: Verify Start, Settings, and About buttons work
2. **Game Flow**: 
   - Start game from boot screen
   - Verify customer request appears
   - Click on books to select
   - Give book to customer
   - Verify educational popup appears
3. **Persistence**: 
   - Play a round
   - Close browser
   - Reopen and verify game state loads
4. **Settings**: Test all settings options
5. **About**: Verify information displays correctly

### Step 8.3: Debug Common Issues
- **Blank screen**: Check browser console for errors
- **State not saving**: Verify localStorage is enabled
- **Styles not loading**: Check CSS file path in `index.html`
- **Navigation issues**: Verify `@page` directives are correct

## Phase 9: Asset Integration

### Step 9.1: Add Assets Folder
```powershell
mkdir Client/wwwroot/assets
mkdir Client/wwwroot/assets/images
mkdir Client/wwwroot/assets/sounds
```

### Step 9.2: Add Images
Place your game assets in `Client/wwwroot/assets/images/`:
- Background images
- Book icons/illustrations
- Character sprites
- UI elements

Reference images in components:
```razor
<img src="/assets/images/book-icon.png" alt="Book" />
```

### Step 9.3: Add Sounds
Place audio files in `Client/wwwroot/assets/sounds/`:
- Background music
- Sound effects (button clicks, page turns, etc.)

Play sounds in components:
```razor
<audio src="/assets/sounds/click.mp3" autoplay />
```

## Phase 10: Build for Production

### Step 10.1: Build Release Version
```powershell
dotnet build -c Release
```

### Step 10.2: Publish
```powershell
dotnet publish -c Release -o publish
```

This creates optimized static files in the `publish` folder.

## Phase 11: Vercel Deployment

### Step 11.1: Prepare for Vercel
1. Copy contents of `Client/bin/Release/net8.0/publish/wwwroot` to a new folder `vercel-deploy`
2. Create `vercel.json` in project root:
```json
{
  "version": 2,
  "builds": [
    {
      "src": "vercel-deploy/**",
      "use": "@vercel/static"
    }
  ]
}
```

### Step 11.2: Deploy to Vercel
```powershell
cd vercel-deploy
vercel
```

Follow the prompts:
- Link to existing project? No
- Project name: tech-library-game
- Directory: . (current directory)
- Settings: Use defaults

### Step 11.3: Configure Environment Variables
In Vercel dashboard:
1. Go to Project Settings
2. Add Environment Variable:
   - Name: `OPENAI_API_KEY`
   - Value: Your OpenAI API key
3. Redeploy

### Step 11.4: Automatic Deployments from GitHub
```powershell
vercel --prod
```

This sets up automatic deployments whenever you push to GitHub.

## Phase 12: GitHub Workflow

### Step 12.1: Initial Commit
```powershell
git add .
git commit -m "Initial commit: Tech Library game setup"
git push origin main
```

### Step 12.2: Development Workflow
For each feature:
```powershell
git checkout -b feature/feature-name
# Make changes
git add .
git commit -m "Add feature description"
git push origin feature/feature-name
# Create pull request on GitHub
```

### Step 12.3: Branch Protection (Recommended)
In GitHub repository settings:
1. Go to Settings > Branches
2. Add rule for `main` branch
3. Require pull request reviews
4. Require status checks to pass

## Phase 13: OpenAI Integration Setup

### Step 13.1: Get OpenAI API Key
1. Go to https://platform.openai.com/
2. Sign up or log in
3. Navigate to API Keys section
4. Create new API key
5. Copy and save securely

### Step 13.2: Configure API Key Locally
For local development, set environment variable:
```powershell
$env:OPENAI_API_KEY="your-api-key-here"
```

Or create `Client/appsettings.Development.json`:
```json
{
  "OpenAI": {
    "ApiKey": "your-api-key-here",
    "Model": "gpt-3.5-turbo"
  }
}
```

Add to `.gitignore`:
```
appsettings.Development.json
```

### Step 13.3: Test OpenAI Integration
Run the app and verify:
- AI-generated content appears in educational popups
- Follow-up questions are generated correctly
- Content varies between sessions

## Phase 14: Advanced Features Implementation

### Step 14.1: Dynamic Difficulty
Update `Game.razor` to adjust difficulty based on `gameState.CustomersServed`:
```csharp
private void AdjustDifficulty()
{
    if (gameState.CustomersServed > 10)
    {
        // Introduce intermediate topics
    }
    if (gameState.CustomersServed > 20)
    {
        // Introduce advanced topics
    }
}
```

### Step 14.2: Scoring System (Optional)
Add scoring logic in `GameStateService`:
```csharp
public async Task AddScore(int points)
{
    gameState.Score += points;
    await SaveGameStateAsync(gameState);
}
```

### Step 14.3: Swipe Navigation
Add touch/swipe support using JavaScript interop:
```csharp
[IJSInvokable]
public void SwipeLeft()
{
    // Navigate to previous book
}

[IJSInvokable]
public void SwipeRight()
{
    // Navigate to next book
}
```

## Phase 15: Documentation and Maintenance

### Step 15.1: Update README.md
The comprehensive `README.md` has already been created in the Context folder. Copy this file to the project root:

```powershell
copy C:\Users\Admin944\Downloads\Context\README.md .\README.md
```

The README.md includes:
- Complete project overview
- System requirements
- Gameplay mechanics
- Game flow with mermaid diagram
- Technical specifications
- Progression systems
- Performance requirements
- Browser compatibility
- Security considerations
- Appendices with glossary and detailed category information

### Step 15.2: Create CHANGELOG.md
Track version changes and features.

### Step 15.3: Regular Maintenance Tasks
- Update dependencies regularly
- Monitor OpenAI API usage
- Collect user feedback
- Add new educational content categories

## Troubleshooting Guide

### Common Issues and Solutions

**Issue: "dotnet command not found"**
- Solution: Verify .NET SDK installation and restart terminal

**Issue: "OpenAI API errors"**
- Solution: Check API key is valid and has credits

**Issue: "Vercel deployment fails"**
- Solution: Ensure publish folder contains all static files

**Issue: "Game state not persisting"**
- Solution: Check browser localStorage is enabled

**Issue: "Styles not loading in production"**
- Solution: Verify CSS paths are relative to wwwroot

## Next Steps After Blueprint

1. Follow each phase sequentially
2. Test thoroughly after each phase
3. Commit changes to GitHub regularly
4. Deploy to Vercel after Phase 11
5. Iterate based on testing feedback
6. Add polish and additional features

## Support and Resources

- Blazor Documentation: https://docs.microsoft.com/aspnet/core/blazor/
- OpenAI API Docs: https://platform.openai.com/docs
- Vercel Docs: https://vercel.com/docs
- .NET Documentation: https://docs.microsoft.com/dotnet/