# MFA002 Microsoft Agent Framework 串流輸出回應

在上一篇關於 Microsoft Agent Framework 系列教學文章 "MFA001 Microsoft Agent Framework 初次體驗開發出 AI 代理" 中，我們已經完成了建立一個簡單 AI 代理的流程，並且成功生成了一篇與鵝有關的詩。

當要透過 Web API 進行呼叫並且取得結果的時候，通常會需要等待整個回應完成後才能取得結果，而無法即時看到生成的內容。但若要回應的內容過多與過大，又或者希望能夠讓只用者即時看到任何 API 處理與回傳的結果時，就需要使用串流輸出回應的方式。對於開發者而言，這種方式可以大幅提升使用者體驗，並且在處理大量資料時更加高效。這兩者間的差異在於是否能即時接收與呈現生成的內容。當然，對於程式碼的撰寫與設計，也需要考慮如何處理串流輸出的資料。

在 C# 程式語言中，面對這兩種的作法，可以分別使用同步呼叫與非同步串流的方式來實現。同步呼叫適合處理回應較小且不需要即時呈現的情境，而非同步串流則適合處理大量資料或需要即時回應的情境。舉個例子來說，同步呼叫就像是一次性地向 API 發送請求，然後等待整個回應完成後才進行後續處理；而非同步串流則是可以在接收到部分回應時就立即進行處理，並持續接收後續的資料，直到整個回應完成。

在這裡的程式碼，將會請求 AI API 來產生出一篇與鵝有關的詩。

這篇文章將會進一步介紹如何透過串流輸出回應的方式，來即時接收 AI 代理生成的內容。

# 建立 Console 專案
* 開啟 Visual Studio 2026
* 選擇「建立新專案」
* 在 [建立新的專案] 視窗中，在右方清單內，找到並選擇「主控台應用程式」 項目
* 然後點擊右下方「下一步」按鈕
* 此時將會看到 [設定新的專案] 對話窗
* 在該對話窗的 [專案名稱] 欄位中，輸入專案名稱，例如 "csStreamAgent"
* 然後點擊右下方「下一步」按鈕
* 接著會看到 [其他資訊] 對話窗
* 在這個對話窗內，確認使用底下的選項
    * 架構：.NET 10.0 (或更新版本)
    * 勾選 不要使用最上層陳述式 (這是我的個人習慣)
* 然後點擊右下方「建立」按鈕
* 現在，已經完成了這個 Console 主控台 專案的建立

## 安裝需要用到的 NuGet 套件

要完成這個第一次接觸 Microsoft Agent Framework 練習專案，需要安裝以下幾個 NuGet 套件：

### Microsoft.Extensions.AI

這個套件將會為生成式 AI 定義的一層「統一抽象介面 + Middleware 管線」，讓開發者可以更方便地整合不同的生成式 AI 模型，並且在應用程式中使用一致的方式進行呼叫與管理。

* 滑鼠右擊 [csStreamAgent] 專案節點
* 點選彈出功能表的 [管理 NuGet 套件] 項
* 在 [瀏覽] 索引標籤中，搜尋並且安裝底下的 NuGet 套件
    * Microsoft.Extensions.AI
    
### Microsoft.Extensions.AI.OpenAI

這個套件是微軟官方為 .NET 整合 AI 服務所提供的擴充套件。它實現了 IChatClient 與 IEmbeddingGenerator 核心抽象介面，讓開發人員能以一致的程式碼，輕鬆串接 OpenAI 或相容的 API 端點（例如 Azure OpenAI、GitHub Models 等）。

* 滑鼠右擊 [csStreamAgent] 專案節點
* 點選彈出功能表的 [管理 NuGet 套件] 項
* 在 [瀏覽] 索引標籤中，搜尋並且安裝底下的 NuGet 套件
    * Microsoft.Extensions.AI.OpenAI
    
### Microsoft.Agents.AI

這個套件是微軟官方推出 Microsoft Agent Framework（微軟智慧代理框架）的核心 .NET 套件。它主要用於在 .NET 環境中建構、協調與部署 AI 代理（AI Agents） 及多代理工作流（Multi-agent workflows），是用來取代舊版 AutoGen 的企業級解決方案。

* 滑鼠右擊 [csStreamAgent] 專案節點
* 點選彈出功能表的 [管理 NuGet 套件] 項
* 在 [瀏覽] 索引標籤中，搜尋並且安裝底下的 NuGet 套件
    * Microsoft.Agents.AI
     
### OpenAI

這個套件為官方的 OpenAI .NET 客戶端類別庫，是 OpenAI NuGet 套件。此套件由 OpenAI 與 Microsoft 合作開發，提供直接存取 OpenAI 官方 REST API 的便利管道。

* 滑鼠右擊 [csStreamAgent] 專案節點
* 點選彈出功能表的 [管理 NuGet 套件] 項
* 在 [瀏覽] 索引標籤中，搜尋並且安裝底下的 NuGet 套件
    * OpenAI

# 宣告 API 端點與 Key 環境變數

若把 AI 用到的 Key 與服務端點曝露在程式碼內，這個程式碼又簽入到版本控制系統（例如 GitHub），將會造成安全風險，因此建議使用環境變數來管理這些敏感資訊。

## Windows 建立環境變數

* 如果是在 Windows 開發環境，可以從「系統環境變數」進行設定。首先，在 Windows 搜尋列輸入「環境變數」，接著開啟：編輯系統環境變數
* 在「系統內容」視窗中選擇：進階→ 環境變數
* 接下來可以選擇建立「使用者變數」或「系統變數」。我個人都是建立 系統變數 的方式。
* 新增第一個變數：
  * 變數名稱：AzureOpenAI_Key
  * 變數值：你的 Azure OpenAI API Key
* 接著再新增第二個變數：
  * 變數名稱：AzureOpenAI_Endpoint
  * 變數值：https://你的資源名稱.openai.azure.com/
* 完成後按下「確定」儲存。

## 使用 PowerShell 建立環境變數

如果習慣使用 PowerShell，也可以直接透過指令建立。例如建立目前 Windows 使用者層級的環境變數：

```powershell
[Environment]::SetEnvironmentVariable(
    "AzureOpenAI_Key",
    "你的 Azure OpenAI API Key",
    "Machine"
)

[Environment]::SetEnvironmentVariable(
    "AzureOpenAI_Endpoint",
    "https://你的資源名稱.openai.azure.com/",
    "Machine"
)
```

設定完成後，通常需要重新開啟 Visual Studio、VS Code、Terminal 或 PowerShell，新的 Process 才會取得最新的環境變數。

# 修改 Program.cs 程式碼

為了要完成這個需求 : 如何用 Azure OpenAI 的 gpt-5.6-luna 聊天模型建立一個簡單 AI 代理，並產生一篇與鵝有關的詩。
因此，我們需要修改 Program.cs 程式碼，這時需要完成底下的操作。

* 在專案內，找到並且開啟 [Program.cs] 檔案
* 使用底下程式碼內容來取代 [Program.cs] 檔案的內容

```csharp
using Microsoft.Agents.AI;
using Microsoft.Extensions.AI;
using OpenAI;
using OpenAI.Chat;
using System.ClientModel;
using System.Diagnostics;

namespace csStreamAgent;

internal class Program
{
    static async Task Main(string[] args)
    {
        var apiKey = Environment.GetEnvironmentVariable("AzureOpenAI_Key");
        var endpoint = Environment.GetEnvironmentVariable("AzureOpenAI_Endpoint");
        var model = "gpt-5.6-luna";

        if (string.IsNullOrWhiteSpace(apiKey) || string.IsNullOrWhiteSpace(endpoint))
        {
            Console.WriteLine("請先設定環境變數 AzureOpenAI_Key 與 AzureOpenAI_Endpoint 後再執行。");
            return;
        }

        IChatClient chatClient =
            new ChatClient(
                    model,
                    new ApiKeyCredential(apiKey),
                    new OpenAIClientOptions { Endpoint = new Uri(endpoint) })
                .AsIChatClient();

        AIAgent agent = new ChatClientAgent(
            chatClient,                      // 聊天用戶端
            "詩人",                          // 代理名稱
            "創作引人入勝、富有創意的詩。.",  // 系統提示詞
            null);                           // 其他設定

        Console.WriteLine($"{DateTime.Now} 開始呼叫 LLM API (串流) / 使用的模型: {model}");

        var stopwatch = Stopwatch.StartNew();
        TimeSpan? timeToFirstToken = null;   
        var chunkCount = 0;                  

        await foreach (var update in agent.RunStreamingAsync("寫一首關於鵝的長詩，至少八段，每段四行。"))
        {
            chunkCount++;

            if (string.IsNullOrEmpty(update.Text))
            {
                continue;   
            }

            timeToFirstToken ??= stopwatch.Elapsed;
            Console.Write(update.Text);  
        }

        stopwatch.Stop();

        Console.WriteLine();
        Console.WriteLine($"{DateTime.Now} 完成呼叫 LLM API (串流) / 使用的模型: {model}");
        Console.WriteLine(
            $"首字延遲: {timeToFirstToken?.TotalMilliseconds ?? 0:F0} ms / " +
            $"總耗時: {stopwatch.Elapsed.TotalMilliseconds:F0} ms / " +
            $"收到片段 chunk 數: {chunkCount}");
    }
}
```

在上述的程式碼中，可以看出當想要使用 Microsoft Agent Framework 套件，開發者需要先初始化聊天用戶端 (IChatClient)，再建立 AI 代理 (AIAgent)，最後透過串流方式呼叫代理並即時取得回應。

因此，開發者只需要依照這個範例程式碼的結構，就能快速建立一個透過串流輸出回應的簡單 AI 代理，並透過 Azure OpenAI 的 gpt-5.6-luna 聊天模型來生成內容。

## 1. 初始化聊天用戶端

* 由於 API Key 是敏感資訊，因此在程式碼中不會直接寫死，而是透過環境變數來取得。
* 在底下程式碼之前，將會透過 `var apiKey = Environment.GetEnvironmentVariable("AzureOpenAI_Key");` & `var endpoint = Environment.GetEnvironmentVariable("AzureOpenAI_Endpoint");` 來取得必要的環境變數。這些環境變數必須要參考前面說明內容，使用 GUI 介面或者 PowerShell 命令，將環境變數與設定值綁定在一起。
  > 注意：在實際開發中，請確保環境變數已正確設定，否則程式將無法成功取得 API Key 與 Endpoint。
* 而模型名稱，則是使用 `var model = "gpt-5.6-luna";` 程式碼，直接寫在程式碼中
  > 這裡選擇的模型是最輕巧、快速回應的模型，僅是作為教學說明目的而採用，你可以根據你本身的需求來設定使用更為強大的模型，
* 但若使用者沒有將 API Key & endpoint 綁定到環境變數內，將會無法成功取得必要的資訊，程式也無法正常執行。
* 所以，在這個練習範例中，將會檢查是否成功取得 API Key 與 Endpoint，若未取得則會提示使用者設定環境變數並終止程式。這裡將會採用底下的程式碼來做到這個檢查。

```csharp
if (string.IsNullOrWhiteSpace(apiKey) || string.IsNullOrWhiteSpace(endpoint))
{
    Console.WriteLine("請先設定環境變數 AzureOpenAI_Key 與 AzureOpenAI_Endpoint 後再執行。");
    return;
}
```

* 接下來，將會用到 Chat Completion 的 API 呼叫模式，因此，在這裡將會需要建立聊天用戶端 (IChatClient)，這裡將會透過new ChatClient(...) 來建立。
* 在建立 ChatClient 執行個體物件時候，將會在建構式中傳入模型名稱、API Key 以及 Endpoint 等必要資訊。
* 最後，透過 AsIChatClient() 方法將 ChatClient 轉型為 IChatClient 介面。
* 在 IChatClient 介面中，將會提供與聊天相關的方法，例如發送訊息、接收回應等，這樣可以讓開發者專注於與 AI 代理的互動，而不需要關心底層的 API 呼叫細節。有了這樣的設計，開發者可以更專注於業務邏輯的實現，而不需要處理繁瑣的 API 呼叫細節。

```csharp
IChatClient chatClient =
    new ChatClient(
            model,
            new ApiKeyCredential(apiKey!),
            new OpenAIClientOptions { Endpoint = new Uri(endpoint) })
        .AsIChatClient();
```

## 2. 建立 AI 代理

有了 IChatClient 物件之後，就可以用它來建立 AI 代理 (AIAgent) 了。這裡使用 ChatClientAgent 作為具體的實現類別，並傳入聊天用戶端、代理名稱、系統提示詞以及其他設定。在這裡，其他設定尚未用到，所以，在這個範例中傳入 null 即可。

* 從底下的程式碼可以看到，建立 AI 代理的過程相對簡單，只需要提供聊天用戶端、代理名稱、系統提示詞以及其他設定即可。這裡使用的系統提示詞為「創作引人入勝、富有創意的詩。」。

```csharp
AIAgent agent = new ChatClientAgent(
    chatClient,                      // 聊天用戶端
    "詩人",                          // 代理名稱
    "創作引人入勝、富有創意的詩。.",  // 系統提示詞
    null);                           // 其他設定
```

* 對於 代理名稱 的用途在於識別不同的 AI 代理，特別是在同一個應用程式中可能會有多個代理同時存在時，代理名稱可以幫助開發者區分不同的代理，並在與代理互動時提供更清晰的上下文。

* 而對於 系統提示詞，則是用來指導 AI 代理的行為與風格，確保生成的內容符合預期。例如在這個範例中，系統提示詞設定為「創作引人入勝、富有創意的詩。」，這樣 AI 代理在生成詩的時候，就會遵循這個指引，創作出符合要求的詩作。對於要能夠學會如何呼叫 LLM API 的開發者來說，理解系統提示詞的作用是非常重要的。除了系統提示詞之外，還會有另外兩類提示詞，分別是用戶提示詞 (User Prompt) 與助手提示詞 (Assistant Prompt)，這些提示詞可以用來進一步引導 AI 代理的行為與回應。

* 所謂使用者提示詞 (User Prompt)，是指開發者或者最終使用者在與 AI 代理互動時所提供的指令或問題，這些提示詞會直接影響 AI 代理的回應內容。使用者提示詞通常用來引導 AI 代理生成特定的內容或完成特定的任務。而助手提示詞 (Assistant Prompt)，則是指 AI 代理在回應使用者提示詞時所使用的提示詞，這些提示詞可以用來控制 AI 代理的回應風格、格式或其他行為。透過合理設計使用者提示詞與助手提示詞，開發者可以更精確地引導 AI 代理的行為，達到預期的互動效果。

* 關於系統提示詞、使用者提示詞與助手提示詞的設計，開發者應該根據具體的應用場景來進行合理的設計，若還不是很了解，可以參考我之前的部落格文章，裡面有更詳細的說明與範例。

## 3. 透過串流方式呼叫代理並即時取得回應

* 為了要了解串流設計表現出來的特性，在這裡使用底下程式碼，將會計算首字延遲以及收到的更新片段數量。

```csharp
var stopwatch = Stopwatch.StartNew();
TimeSpan? timeToFirstToken = null;   // 首字延遲：送出請求到收到第一段文字
var chunkCount = 0;                  // 總共收到幾個更新片段
```

* 現在已經取得了 AIAgent 物件，可以使用它來與 AI 代理進行互動，並取得回應。
為了要能夠使用串流作業方式，這裡將會使用 RunStreamingAsync 方法來發送使用者提示詞，並即時取得 AI 代理的回應。
* 另外，這裡使用 C# 的 await foreach 語法來處理串流回應，這樣可以在收到每一段更新時即時進行處理，而不需要等到整個回應完成後才進行處理。這樣設計方式的好處與傳統的做法相比，可以顯著降低首字延遲，提升使用者體驗。這樣的語法將會是在 C# 8.0 之後引入的。
* 在這個範例中，使用者提示詞為 "寫一首關於鵝的長詩，至少八段，每段四行。"，這樣 AI 代理就會根據這個提示詞來生成對應的詩作。會故意這樣設計，是要避免產生的結果文字過短，看不出串流的效果。
* 在 await foreach 迴圈中，每次收到的更新都會被處理，這樣可以即時顯示 AI 代理生成的內容，而不需要等到整個回應完成。
* 當 update.Text 為空時，表示這個更新只包含中繼資料而沒有文字內容，因此會被跳過。
* timeToFirstToken 代表從發送請求到收到第一段文字的時間，也就是首字延遲。這個指標可以用來衡量串流回應的即時性，對於需要快速回應的應用場景非常重要。
* chunkCount 則是用來統計總共收到幾個更新片段，這個指標可以用來了解串流回應的分段情況，對於分析回應的即時性與完整性有幫助。
* 一旦收到一段文字之後，將會使用 Console.Write 將文字即時輸出到控制台，這樣使用者就可以立即看到 AI 代理生成的內容。

```csharp
await foreach (var update in agent.RunStreamingAsync("寫一首關於鵝的長詩，至少八段，每段四行。"))
{
    chunkCount++;

    if (string.IsNullOrEmpty(update.Text))
    {
        continue;   // 部分更新只帶中繼資料、沒有文字內容
    }

    timeToFirstToken ??= stopwatch.Elapsed;
    Console.Write(update.Text);   // 用 Write 不換行，讓文字接續湧出
}
```

# 執行結果

現在來看看這個範例程式碼的執行結果。由於使用了串流方式，文字會逐步湧出，而不是一次性全部顯示。

* 在 Visual Studio 2026 下，按下 F5 鍵執行程式，將會看到程式輸出的結果。

```plaintext
2026/10/6 下午 02:53:22 開始呼叫 LLM API (串流) / 使用的模型: gpt-5.6-luna
## 《鵝在水邊寫下的長詩》

一
清晨的河面還含著霧，
一隻白鵝從蘆葦間醒來，
牠把長頸伸向初升的太陽，
像替大地拉開一幅潔白的帆。

二
紅掌輕輕撥開薄薄的水，
波紋便向遠方一圈圈散去，
牠的影子沉在碧綠深處，
彷彿雲朵也學會了游泳。

三
牠走上岸時，風忽然安靜，
草葉在腳邊低頭致意，
每一步都帶著莊嚴的回聲，
像古老王國巡行的使者。

四
孩子們追著牠歡笑奔跑，
手裡握著未完成的紙船，
鵝回頭高聲說了一句話，
沒有人懂，卻都覺得春天來了。

五
午後的天空掛滿藍色，
牠伏在柳蔭下整理羽毛，
一片一片，梳理自己的夢，
也梳理河流遺落的星光。

六
當稻田被夕陽染成金色，
鵝群排成長長的行列，
牠們穿過田埂與斜陽，
像一串移動的白色標點。

七
夜裡，月亮落進池塘，
鵝把清冷的月光守在胸前，
遠處的蛙聲一遍遍歌唱，
近處的水面悄悄替牠鼓掌。

八
有時風暴從北方趕來，
黑雲壓低了高遠的天，
鵝仍昂起那不屈的長頸，
在雷聲裡喊出自己的名字。

九
牠不是王，也不佩戴冠冕，
卻懂得守護一方水草，
懂得在迷失的同伴身旁，
用一聲鳴叫照亮歸途。

十
冬天結霜，河岸變得蒼白，
鵝的腳印留在薄薄的雪上，
一深一淺，倔強而清楚，
像時間寫給大地的信。

十一
春水再次漫過低矮的橋，
去年凋落的蘆花重新發芽，
那隻白鵝抬頭望向遠方，
眼中藏著無數尚未抵達的湖泊。

十二
於是牠展開寬闊的翅膀，
並非為了逃離這片故鄉，
而是讓每一個平凡的早晨，
都知道天空仍值得仰望。
2026/10/6 下午 02:53:29 完成呼叫 LLM API (串流) / 使用的模型: gpt-5.6-luna
首字延遲: 3172 ms / 總耗時: 6889 ms / 收到片段 chunk 數: 603
```

從執行結果可以看出，文字是逐步湧出的，而不是一次性全部顯示，這說明串流回應的設計確實能夠降低首字延遲，提升使用者體驗。同時，首字延遲和總耗時的統計也能幫助我們了解整個回應過程的即時性與效率。另外從 chunk 數: 603 可以看出，整個回應被分成了多個片段，這也說明了串流回應的分段特性。

