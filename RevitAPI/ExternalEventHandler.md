# 01
Revit IExternalEventHandler 有什么作用
<details style="color: #999">
    <summary style="color: #666; font-size: 12px">思考</summary>
    
好的，我现在需要回答用户关于Revit中IExternalEventHandler的作用的问题。首先，我得回忆一下自己对Revit API的了解，特别是关于外部事件处理的部分。

用户可能是一个刚开始学习Revit二次开发的开发者，或者在使用过程中遇到了需要处理异步操作的情况。他们可能想知道为什么需要使用这个接口，以及它在实际开发中的具体应用场景。

首先，我应该明确IExternalEventHandler的基本作用。根据之前的知识，Revit的API操作通常需要在主线程执行，而某些操作，特别是那些需要用户交互或长时间运行的任务，可能需要异步处理以避免阻塞UI。这时候，IExternalEventHandler就派上用场了，它允许开发者将代码的执行延迟到合适的时间，由Revit的内部事件机制来触发。

接下来，我需要详细解释这个接口的主要用途。比如，当执行一个需要用户交互（如弹窗选择文件）的操作时，直接在主线程执行可能会导致问题。使用IExternalEventHandler可以将这些操作封装起来，在外部事件处理程序中执行，确保线程安全。

然后，我应该提到接口的关键方法Execute和属性Name。Execute方法是实际执行操作的地方，而Name属性用于调试或日志记录，帮助识别不同的事件处理器。

此外，可能还需要说明如何实际使用这个接口。例如，创建一个实现该接口的类，然后在需要的时候通过ExternalEvent.Create方法创建ExternalEvent实例，并调用Raise方法来触发事件。

用户可能还想知道使用这个接口的好处，比如避免UI冻结、处理模态对话框时的限制，或者确保API调用在正确的上下文中执行。这些都是需要涵盖的点。

另外，可能需要举一个简单的例子，比如在按钮点击事件中触发一个外部事件，然后在Execute方法中执行具体的Revit操作，如修改模型或显示对话框。

还要注意可能的相关问题，比如ExternalEvent类和IExternalEvent接口之间的关系，以及为什么不能直接在非UI线程中调用Revit API。这里需要强调Revit API的线程模型要求大部分API调用必须在主线程执行，而外部事件处理机制正是为了满足这一要求而设计的。

最后，确保解释清晰，结构合理，用简洁的语言让用户能够理解IExternalEventHandler的作用和使用方法，同时覆盖常见的使用场景和注意事项。
</details>

<details>
    <summary style="color: #666; font-size: 12px">正文</summary>

在Revit API中，**`IExternalEventHandler`** 是一个用于处理异步操作和线程间调用的关键接口。它的主要作用是帮助开发者在**Revit主线程上下文**中安全地执行代码，尤其是在需要与用户交互或执行耗时操作时，避免阻塞Revit的UI线程或违反API的线程限制。

---

### **主要作用**
1. **异步执行API操作**  
   Revit API严格要求大多数操作必须在主线程执行（例如修改模型、访问文档数据）。  
   `IExternalEventHandler` 允许你将代码逻辑封装到一个事件处理器中，并通过**外部事件队列**异步触发，确保这些操作在正确的线程上下文中执行。

2. **处理用户交互**  
   当你的插件需要弹出对话框（如`OpenFileDialog`）或等待用户输入时，直接在主线程调用这些操作可能导致Revit无响应。通过`IExternalEventHandler`，可以将这类操作推迟到安全的时间点执行。

3. **避免UI冻结**  
   长时间运行的任务（如遍历大量元素、生成复杂几何体）如果直接在主线程运行，会导致Revit界面冻结。使用此接口可以将任务拆分或延迟执行，保持UI响应。

4. **兼容模态操作**  
   在Revit的模态状态下（例如用户正在编辑草图），某些API操作会被禁止。通过外部事件，可以绕过这些限制，选择合适时机执行代码。

---

### **核心方法**
- **`Execute(UIApplication app)`**  
  必须实现的方法，包含实际要执行的逻辑。当外部事件被触发时，Revit会在此方法内运行你的代码。

- **`string GetName()`**  
  返回事件处理器的名称（用于调试和日志）。

---

### **使用流程**
1. **创建事件处理器类**  
   实现`IExternalEventHandler`接口：
   ```csharp
   public class MyEventHandler : IExternalEventHandler
   {
       public void Execute(UIApplication app)
       {
           // 在这里写需要执行的Revit API操作
           TaskDialog.Show("提示", "Hello from Execute!");
       }

       public string GetName()
       {
           return "My Custom Event Handler";
       }
   }
   ```

2. **创建`ExternalEvent`实例**  
   在插件初始化时（如`OnStartup`），创建事件对象：
   ```csharp
   private ExternalEvent _myEvent;
   private MyEventHandler _handler;

   _handler = new MyEventHandler();
   _myEvent = ExternalEvent.Create(_handler);
   ```

3. **触发事件**  
   在需要的地方（如按钮点击事件）调用：
   ```csharp
   _myEvent.Raise();
   ```
   Revit会在下一个空闲时间点调用`Execute`方法。

---

### **典型场景**
- 用户点击按钮后弹出一个对话框选择文件，然后根据文件内容修改模型。
- 在后台执行耗时计算（如结构分析），完成后更新模型。
- 处理其他线程（如Task或Thread）中需要调用Revit API的操作。

---

### **注意事项**
- **线程安全**：`Execute`方法始终在Revit主线程执行，无需手动同步。
- **频率控制**：频繁调用`Raise()`可能导致事件队列堆积，需合理设计逻辑。
- **生命周期管理**：确保`ExternalEvent`实例在插件生命周期内有效（通常作为类成员变量）。

通过`IExternalEventHandler`，开发者可以更灵活地协调Revit API调用与用户交互，提升插件的稳定性和用户体验。
</details>

---

# 02
如何管理或设计其生命周期？
<details style="color: #999">
    <summary style="color: #666; font-size: 12px">思考</summary>
    
嗯，用户之前问了关于Revit的IExternalEventHandler有什么作用，现在又接着问如何管理或设计其生命周期。看来他们可能正在开发一个Revit插件，遇到了多线程或异步处理的问题，需要更深入的指导。用户可能已经了解了基本用法，现在想确保他们的实现是稳定和高效的。

首先，我需要回顾一下IExternalEventHandler的生命周期管理。生命周期管理的关键在于正确创建、使用和销毁ExternalEvent实例，避免内存泄漏或意外行为。用户可能关心的问题包括：何时创建ExternalEvent，如何确保它在需要时可用，如何处理多次触发，以及如何在插件卸载时正确清理资源。

接下来，用户可能是在实际开发中遇到了问题，比如事件处理器没有被正确触发，或者Revit崩溃，这可能是因为生命周期管理不当。例如，如果ExternalEvent实例被意外销毁，或者在错误的时机被调用，就会导致问题。需要强调将ExternalEvent作为类成员变量，保持其在整个插件生命周期中的有效性。

另外，用户可能想知道如何处理多次触发事件的情况。比如，用户多次点击按钮触发事件，是否需要队列机制或者防止重复执行。这时候，可以讨论ExternalEvent的Pending状态，以及如何通过禁用按钮或者检查状态来避免堆积。

还有，用户可能关心如何在不同的Revit文档或会话中管理事件处理器。比如，当用户切换文档时，是否需要重新创建ExternalEvent实例。需要说明ExternalEvent与特定文档无关，但需要注意文档状态的变化，比如在事件处理中检查文档是否仍然有效。

此外，用户可能对异步操作中的异常处理不太清楚，需要指导如何在Execute方法中捕获异常，避免导致Revit崩溃。同时，可能需要提醒在事件处理器中避免长时间运行的操作，以免阻塞主线程，尽管已经通过异步事件处理，但长时间任务仍需拆分。

最后，用户可能希望看到具体的代码示例和最佳实践，比如使用单例模式或依赖注入来管理事件处理器，或者如何在WPF窗口中集成ExternalEvent的触发。需要提供结构清晰的代码示例，并解释每一步的考虑因素，比如为什么要在OnStartup中初始化，如何在Dispose中释放资源。

总结下来，用户的需求不仅仅是了解生命周期管理的步骤，而是希望确保他们的实现是健壮的、高效的，并且能够处理各种边界情况，比如多次触发、资源释放、异常处理等。需要从实例化、使用、状态管理、资源清理等多个方面详细展开，同时结合实际场景和常见问题，给出具体的解决方案和最佳实践。
</details>

<details>
    <summary style="color: #666; font-size: 12px">正文</summary>

在Revit插件开发中，合理管理`IExternalEventHandler`及其关联的`ExternalEvent`实例的生命周期至关重要，这直接关系到插件的稳定性、资源占用和用户体验。以下是详细的设计和管理方法：

---

### **1. 生命周期管理核心原则**
- **持久性**：`ExternalEvent`实例需长期存活（通常与插件主类同生命周期）。
- **线程安全**：始终通过`ExternalEvent`机制触发主线程操作，避免直接跨线程调用。
- **资源释放**：在插件卸载时正确清理资源，防止内存泄漏。

---

### **2. 生命周期设计步骤**

#### **2.1 实例化阶段**
- **创建位置**：在插件初始化时（如`IExternalApplication.OnStartup`）或首次需要时创建。
- **推荐模式**：使用单例模式或依赖注入确保全局唯一性。

```csharp
public class MyPlugin : IExternalApplication
{
    private ExternalEvent _externalEvent;
    private MyEventHandler _eventHandler;

    public Result OnStartup(UIControlledApplication app)
    {
        // 初始化事件处理器和ExternalEvent
        _eventHandler = new MyEventHandler();
        _externalEvent = ExternalEvent.Create(_eventHandler);
        return Result.Succeeded;
    }
}
```

#### **2.2 使用阶段**
- **触发事件**：通过`Raise()`方法请求执行，但需注意频率控制。
- **状态监控**：利用`ExternalEvent.Status`属性判断事件状态：

| 状态 (`ExternalEvent.Status`) | 描述 |
|-------------------------------|------|
| `Pending`                     | 事件已加入队列，等待执行 |
| `NotAvailable`                | 事件未被创建或已被销毁 |
| `Succeeded`/`Failed`          | 最近一次执行结果 |

```csharp
// 示例：安全触发事件
if (_externalEvent.Status == ExternalEventStatus.Pending)
{
    TaskDialog.Show("警告", "已有事件待处理！");
}
else
{
    _externalEvent.Raise();
}
```

#### **2.3 资源释放阶段**
- **销毁时机**：在插件关闭时（如`IExternalApplication.OnShutdown`）或明确不再需要时销毁。
- **显式释放**：调用`Dispose()`（尽管Revit可能自动处理，但显式释放更安全）。

```csharp
public Result OnShutdown(UIControlledApplication app)
{
    // 显式释放资源
    _externalEvent.Dispose();
    _eventHandler = null;
    return Result.Succeeded;
}
```

---

### **3. 高级生命周期管理技巧**

#### **3.1 防止重复触发**
- **UI按钮禁用**：在事件处理期间禁用触发按钮：
```csharp
// 在Execute方法中控制UI状态
public void Execute(UIApplication app)
{
    try
    {
        // 禁用按钮
        MyRibbonButton.Enabled = false;
        
        // 核心逻辑...
    }
    finally
    {
        // 重新启用按钮
        MyRibbonButton.Enabled = true;
    }
}
```

- **状态标志位**：使用`bool isProcessing`标志阻止并发执行。

#### **3.2 异常处理**
- **全局异常捕获**：在`Execute`方法中包裹`try-catch`：
```csharp
public void Execute(UIApplication app)
{
    try
    {
        // 可能抛出异常的操作
        doc.Delete(elementId);
    }
    catch (Exception ex)
    {
        TaskDialog.Show("错误", $"操作失败: {ex.Message}");
        Logger.LogError(ex);
    }
}
```

#### **3.3 跨文档安全**
- **文档状态验证**：在执行操作前检查文档是否有效：
```csharp
public void Execute(UIApplication app)
{
    var doc = app.ActiveUIDocument?.Document;
    if (doc == null || doc.IsReadOnly)
    {
        TaskDialog.Show("错误", "文档不可用或只读！");
        return;
    }
    
    // 安全操作...
}
```

#### **3.4 异步协同**
- **与Task配合**：将耗时计算放在后台线程，仅通过事件处理器更新Revit模型：
```csharp
public void Execute(UIApplication app)
{
    // 触发后台任务
    Task.Run(() =>
    {
        var result = HeavyCalculation();
        
        // 通过事件处理器更新UI
        Application.Current.Dispatcher.Invoke(() =>
        {
            _externalEvent.Raise(); // 触发另一个事件更新模型
        });
    });
}
```

---

### **4. 典型生命周期管理架构**

#### **4.1 集中式事件管理器**
```csharp
public class EventManager : IDisposable
{
    private readonly Dictionary<string, ExternalEvent> _events = new();

    public void RegisterHandler(string key, IExternalEventHandler handler)
    {
        if (!_events.ContainsKey(key))
        {
            _events[key] = ExternalEvent.Create(handler);
        }
    }

    public void RaiseEvent(string key)
    {
        if (_events.TryGetValue(key, out var evt) && evt.Status != ExternalEventStatus.Pending)
        {
            evt.Raise();
        }
    }

    public void Dispose()
    {
        foreach (var evt in _events.Values)
        {
            evt.Dispose();
        }
        _events.Clear();
    }
}
```

#### **4.2 使用示例**
```csharp
// 初始化
var eventManager = new EventManager();
eventManager.RegisterHandler("ModelUpdate", new ModelUpdateHandler());

// 触发事件
eventManager.RaiseEvent("ModelUpdate");

// 销毁
eventManager.Dispose();
```

---

### **5. 常见问题与解决方案**

| 问题现象 | 原因 | 解决方案 |
|---------|------|----------|
| 点击按钮无响应 | `ExternalEvent`未正确初始化 | 确保在插件启动时创建实例 |
| Revit卡死 | 在`Execute`中执行耗时操作 | 拆分任务，使用`Task`+多事件处理器 |
| 内存泄漏 | 未调用`Dispose()` | 显式释放资源，尤其在动态创建事件时 |
| 事件重复执行 | 频繁调用`Raise()` | 检查`ExternalEvent.Status`或添加标志位 |

---

### **6. 最佳实践总结**
1. **单一职责原则**：每个事件处理器只处理一种类型的操作。
2. **早初始化晚释放**：在插件启动时创建，关闭时销毁。
3. **防御性编程**：始终验证文档状态和事件状态。
4. **日志记录**：在关键节点记录事件触发和执行情况。
5. **压力测试**：模拟高频次事件触发验证稳定性。

通过严谨的生命周期管理，可确保基于`IExternalEventHandler`的插件在复杂场景下仍能保持高效稳定运行。
</details>

---

# 03
再深入讲讲与Task配合的要点
<details style="color: #999">
    <summary style="color: #666; font-size: 12px">思考</summary>
    
好的，用户之前已经询问了关于Revit API中的IExternalEventHandler的作用以及如何管理其生命周期，现在他们希望更深入地了解如何将IExternalEventHandler与Task配合使用。我需要详细解释在异步编程中如何安全地结合这两者，尤其是在Revit的环境中。

首先，我需要回顾之前的内容，确保用户已经理解IExternalEventHandler的基本用法和生命周期管理。然后，重点转移到与Task的配合。可能用户在实际开发中遇到了性能问题，需要在后台执行耗时操作，同时保持UI响应，并最终在主线程更新Revit模型。这时候，Task可以用来处理后台计算，而IExternalEventHandler则负责将结果传回主线程。

接下来，需要考虑Revit API的线程限制。Revit的API大多数操作必须在主线程执行，因此在后台Task中不能直接调用这些API。需要明确划分哪些代码在Task中运行，哪些必须通过IExternalEventHandler在主线程执行。

用户可能想了解如何正确地在Task中启动异步操作，然后在完成后触发事件处理器。需要强调不能在Task中直接调用Revit API，必须通过Raise()方法将更新操作排入主线程队列。

另外，可能需要处理异步操作的异常，避免整个应用程序崩溃。还要考虑如何同步数据，确保后台线程和主线程之间的数据传递是线程安全的，比如使用锁或者其他同步机制。

用户可能还会关心如何优化性能，避免频繁地创建和销毁Task，或者如何管理多个并发的Task。这时候可以讨论使用任务调度器、取消令牌（CancellationToken）来控制任务的执行和取消。

另外，用户可能想知道如何处理长时间运行的任务，比如显示进度条或允许用户取消操作。这时候需要结合Task和IExternalEventHandler来实现进度更新和取消请求的处理。

还要提到一些常见的陷阱，比如在Task中直接修改Revit模型导致崩溃，或者忘记处理跨线程的UI更新。需要提供具体的代码示例，展示正确的模式，比如在后台计算完成后，通过事件处理器来执行Revit API调用。

最后，总结最佳实践，强调分离计算逻辑和Revit操作，使用事件处理器作为桥梁，确保线程安全，以及合理的异常处理和资源管理。这样用户能够全面理解如何有效结合Task和IExternalEventHandler，提升插件的性能和用户体验。
</details>

<details>
    <summary style="color: #666; font-size: 12px">正文</summary>

在 Revit 插件开发中，将 **`IExternalEventHandler`** 与 **`Task`** 结合使用，可以实现后台异步计算与主线程安全操作的完美协同。这种组合尤其适用于需要**长时间运行的计算**（如结构分析、数据生成）或**跨线程数据同步**的场景。以下是深入的技术要点和实践指南：

---

### **1. 核心设计原则**
- **职责分离**：  
  `Task` 负责**纯计算逻辑**（无Revit API调用），`IExternalEventHandler` 负责**主线程操作**（更新模型、UI交互）。
- **线程安全**：  
  **Revit API 只能在主线程调用**，后台线程（如`Task`）只能准备数据，最终通过`Raise()`触发主线程操作。
- **数据隔离**：  
  跨线程传递数据需通过**线程安全容器**（如 `ConcurrentQueue`、`BlockingCollection`）或**深拷贝**。

---

### **2. 基本协作流程**
#### **步骤 1：启动后台任务**
```csharp
public class CalculationHandler : IExternalEventHandler
{
    public List<double> Results { get; private set; } // 存储计算结果

    public void Execute(UIApplication app)
    {
        // 主线程中启动后台任务
        Task.Run(() => HeavyCalculation(app));
    }

    private void HeavyCalculation(UIApplication app)
    {
        // 纯计算逻辑（无Revit API）
        var tempResults = new List<double>();
        for (int i = 0; i < 1000000; i++)
        {
            tempResults.Add(Math.Sqrt(i));
        }

        // 将结果存入线程安全容器
        Results = new List<double>(tempResults);

        // 触发另一个事件处理器更新模型
        UpdateModelEvent.Raise();
    }
}
```

#### **步骤 2：通过事件处理器更新模型**
```csharp
public class UpdateModelHandler : IExternalEventHandler
{
    public void Execute(UIApplication app)
    {
        // 在主线程中安全使用Revit API
        var doc = app.ActiveUIDocument.Document;
        using (Transaction tx = new Transaction(doc, "Update Model"))
        {
            tx.Start();
            // 使用CalculationHandler.Results更新模型...
            tx.Commit();
        }
    }
}
```

---

### **3. 关键技术细节**

#### **3.1 异步上下文管理**
- **`TaskScheduler` 同步**：  
  若需在后台任务中访问 UI 元素（非Revit API），需切换至主线程上下文（如 WPF 的 `Dispatcher`）：
```csharp
Task.Run(() =>
{
    // 后台计算...
    Application.Current.Dispatcher.Invoke(() =>
    {
        // 更新WPF控件（非Revit UI）
        ProgressBar.Value = 100;
    });
});
```

- **避免 `async/await` 陷阱**：  
  `async` 方法默认会尝试在原始上下文恢复执行，需明确指定 `ConfigureAwait(false)`：
```csharp
public async void Execute(UIApplication app)
{
    await Task.Run(() => HeavyCalculation())
              .ConfigureAwait(false); // 禁止尝试返回主线程

    // 此处仍在后台线程！
    // 必须通过 Raise() 触发主线程操作
    _updateEvent.Raise();
}
```

#### **3.2 进度反馈机制**
- **线程安全进度报告**：  
  使用 `IProgress<T>` 接口实现跨线程进度更新：
```csharp
public class CalculationHandler : IExternalEventHandler
{
    private readonly IProgress<int> _progress;

    public CalculationHandler(IProgress<int> progress)
    {
        _progress = progress;
    }

    public void Execute(UIApplication app)
    {
        Task.Run(() =>
        {
            for (int i = 0; i < 100; i++)
            {
                // 计算...
                _progress.Report(i);
            }
        });
    }
}

// 在UI层（如WPF）：
var progress = new Progress<int>(percent =>
{
    ProgressBar.Value = percent;
});
var handler = new CalculationHandler(progress);
```

#### **3.3 取消长时间任务**
- **`CancellationToken` 集成**：  
  允许用户主动取消后台计算：
```csharp
private CancellationTokenSource _cts;

public void Execute(UIApplication app)
{
    _cts = new CancellationTokenSource();
    Task.Run(() =>
    {
        for (int i = 0; i < 100; i++)
        {
            if (_cts.Token.IsCancellationRequested)
            {
                // 清理资源并退出
                return;
            }
            // 计算...
        }
    }, _cts.Token);
}

// 在取消按钮事件中：
_cts?.Cancel();
```

---

### **4. 高级模式：管道式处理**
适用于多阶段异步任务（如数据获取 → 计算 → 模型更新）。

#### **架构设计**
```csharp
public class PipelineManager
{
    private readonly ExternalEvent _dataFetchEvent;
    private readonly ExternalEvent _calculateEvent;
    private readonly ExternalEvent _updateEvent;

    public PipelineManager()
    {
        // 初始化三个事件处理器
        _dataFetchEvent = ExternalEvent.Create(new DataFetchHandler());
        _calculateEvent = ExternalEvent.Create(new CalculateHandler());
        _updateEvent = ExternalEvent.Create(new UpdateHandler());
    }

    public void StartPipeline()
    {
        _dataFetchEvent.Raise();
    }
}

// 数据获取处理器
public class DataFetchHandler : IExternalEventHandler
{
    public void Execute(UIApplication app)
    {
        Task.Run(() =>
        {
            // 后台获取数据...
            _calculateEvent.Raise(); // 触发下一步
        });
    }
}

// 计算处理器
public class CalculateHandler : IExternalEventHandler
{
    public void Execute(UIApplication app)
    {
        Task.Run(() =>
        {
            // 后台计算...
            _updateEvent.Raise(); // 触发更新
        });
    }
}
```

---

### **5. 性能优化技巧**

#### **5.1 任务分块（Chunking）**
将大数据集拆分为小块，定期返回主线程更新进度：
```csharp
public void Execute(UIApplication app)
{
    Task.Run(() =>
    {
        var allElements = GetLargeElementList(); // 获取10万个元素
        const int chunkSize = 1000;
        
        for (int i = 0; i < allElements.Count; i += chunkSize)
        {
            var chunk = allElements.Skip(i).Take(chunkSize).ToList();
            
            // 处理当前块...
            
            // 定期触发UI更新
            if (i % 5000 == 0)
            {
                _progressEvent.Raise();
            }
        }
    });
}
```

#### **5.2 内存管理**
- **及时释放非托管资源**：  
  在 `Task` 中使用的临时文件、数据库连接等需显式释放。
- **避免大对象驻留**：  
  使用 `WeakReference` 或定期清理缓存。

---

### **6. 调试与监控**

#### **6.1 跨线程调试**
- **Visual Studio 线程窗口**：  
  监控后台线程状态。
- **日志标记线程ID**：  
```csharp
Debug.WriteLine($"当前线程: {Thread.CurrentThread.ManagedThreadId}");
```

#### **6.2 性能分析**
- **使用 `Stopwatch`**：  
  测量关键代码段耗时。
- **Revit 内置日志**：  
  通过 `Application.WriteJournalEntry` 记录事件触发时间。

---

### **7. 典型问题解决方案**

| 问题场景 | 解决方案 |
|---------|----------|
| 后台任务修改Revit模型崩溃 | 确保所有模型操作通过`IExternalEventHandler`执行 |
| 进度条更新延迟 | 使用 `IProgress<T>` + `Report` 高频更新 |
| 用户取消操作后数据不一致 | 实现事务回滚机制 |
| 内存泄漏 | 使用 `using` 块管理 `IDisposable` 对象 |

---

### **8. 完整示例：异步导入数据并生成墙体**
```csharp
// 主事件处理器
public class ImportDataHandler : IExternalEventHandler
{
    private readonly ExternalEvent _generateWallsEvent;
    public List<Line> WallLines { get; private set; }

    public ImportDataHandler(ExternalEvent generateWallsEvent)
    {
        _generateWallsEvent = generateWallsEvent;
    }

    public void Execute(UIApplication app)
    {
        Task.Run(() =>
        {
            // 后台读取外部文件（无API调用）
            var lines = File.ReadAllLines("data.csv")
                           .Select(ParseLineToGeometry)
                           .ToList();

            WallLines = lines;

            // 触发生成墙体事件
            _generateWallsEvent.Raise();
        });
    }
}

// 墙体生成处理器
public class GenerateWallsHandler : IExternalEventHandler
{
    public void Execute(UIApplication app)
    {
        var doc = app.ActiveUIDocument.Document;
        using (Transaction tx = new Transaction(doc, "Create Walls"))
        {
            tx.Start();
            foreach (var line in _importHandler.WallLines)
            {
                Wall.Create(doc, line, LevelId, false);
            }
            tx.Commit();
        }
    }
}
```

---

通过合理利用 `Task` 与 `IExternalEventHandler` 的协作，开发者可以构建出**响应迅速、稳定可靠**的 Revit 插件，同时充分利用多核 CPU 的计算能力。关键在于始终遵守 Revit API 的线程规则，并通过清晰的分层架构实现计算逻辑与模型操作的解耦。
</details>

---

# 00
问题
<details style="color: #999">
    <summary style="color: #666; font-size: 12px">思考</summary>
    

</details>

<details>
    <summary style="color: #666; font-size: 12px">正文</summary>


</details>

---
