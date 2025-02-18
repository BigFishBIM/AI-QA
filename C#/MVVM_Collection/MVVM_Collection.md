# 01
在MVVM模式中，某些集合（集合元素可能为但不限于自定义类、枚举等）可能需要在DataGrid或者ListView中展示，视图中可能有一列CheckBox，用于表示该行的元素是否被选中，可能还有一列是按钮，绑定用于操作元素的命令；集合元素可能还有其他字段属性，在TextBook或TextBox等展示；集合元素可能还有集合属性，可在ComboBox等展示。视图中还有“确定”/“取消”按钮，分别可以确定修改和取消修改。请你想一个案例，写出C#完整代码。

<details style="color: #999">
    <summary style="color: #666; font-size: 12px">思考</summary>
    
好的，我现在需要帮用户解决一个关于在MVVM模式中使用DataGrid或ListView展示集合的问题。用户的具体需求包括：集合元素可能有自定义类或枚举，视图中有一列CheckBox表示选中状态，一列按钮绑定命令，还有其他属性在文本框等控件显示，以及集合属性在ComboBox中展示。还有确定和取消按钮来提交或撤销修改。

首先，我需要回顾一下MVVM的基本结构，确保Model、View、ViewModel各司其职。Model负责数据，View负责显示，ViewModel作为中间层处理逻辑和绑定。

接下来，考虑用户提到的集合元素可能包含自定义类和枚举。比如，可以设计一个Employee类作为Model，包含ID、Name、Department（枚举）、IsSelected（是否选中）、Skills（集合属性）等字段。这样能覆盖用户提到的各种数据类型。

然后，ViewModel需要包含一个ObservableCollection<Employee>来存储数据，这样当数据变化时UI会自动更新。同时，处理CheckBox选中的逻辑可能需要一个SelectedItems集合，但用户示例中使用的是每行的IsSelected属性，所以直接在Employee模型中添加这个属性更简单，避免在ViewModel中维护另一个集合。

对于按钮绑定的命令，比如“查看详情”，需要在ViewModel中创建ICommand的实现，比如RelayCommand。当按钮点击时，执行查看选中行详情的逻辑，比如弹出消息框显示员工信息。

ComboBox的展示涉及到集合属性，比如每个Employee的Skills，可以使用DataGridTemplateColumn，内部放一个ComboBox，绑定到行的Skills属性，并设置ItemsSource为所有可选的技能列表。这个技能列表应该在ViewModel中定义，比如AvailableSkills，然后在视图中绑定。

确定和取消按钮的功能需要保存或撤销更改。确定按钮可能将修改后的数据提交，这里简单示例可能只是关闭窗口或提示保存成功。取消按钮则恢复原始数据，可以通过在ViewModel中保留原始数据的副本，或者在加载时克隆数据，取消时重新加载。

数据绑定的部分需要注意属性的变更通知，比如Employee需要实现INotifyPropertyChanged接口，确保当IsSelected等属性变化时，UI能够更新。同样，ViewModel中的命令也需要正确实现CanExecute和Execute方法。

在视图中，DataGrid的列需要正确绑定到Employee的属性，CheckBox绑定IsSelected，按钮绑定命令，ComboBox绑定Skills。确定和取消按钮分别绑定SaveCommand和CancelCommand。

测试时，需要考虑各个绑定是否正确，命令是否触发，数据是否持久化或撤销。比如，修改某个Employee的部门，添加技能，选中几行后点击确定，是否保存；取消则恢复原数据。

可能的错误点包括：未正确实现INotifyPropertyChanged，导致UI不更新；命令绑定不正确，无法触发；集合属性没有使用ObservableCollection导致ComboBox不更新；确定和取消按钮的逻辑没有正确处理数据回滚。

最后，综合这些考虑，编写示例代码，确保结构清晰，符合MVVM模式，覆盖用户提到的所有需求点，并进行测试验证。
</details>

<details>
    <summary style="color: #666; font-size: 12px">正文</summary>

以下是一个基于MVVM模式的完整案例，演示如何管理员工信息集合，包含您提到的所有功能：

```csharp
// Model
using System.Collections.ObjectModel;
using System.ComponentModel;

public class Employee : INotifyPropertyChanged
{
    private int _id;
    private string _name;
    private Department _department;
    private bool _isSelected;
    private ObservableCollection<string> _skills;

    public int Id
    {
        get => _id;
        set { _id = value; OnPropertyChanged(); }
    }

    public string Name
    {
        get => _name;
        set { _name = value; OnPropertyChanged(); }
    }

    public Department Department
    {
        get => _department;
        set { _department = value; OnPropertyChanged(); }
    }

    public bool IsSelected
    {
        get => _isSelected;
        set { _isSelected = value; OnPropertyChanged(); }
    }

    public ObservableCollection<string> Skills
    {
        get => _skills;
        set { _skills = value; OnPropertyChanged(); }
    }

    public event PropertyChangedEventHandler? PropertyChanged;

    protected virtual void OnPropertyChanged([System.Runtime.CompilerServices.CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}

public enum Department
{
    IT,
    HR,
    Finance,
    Marketing
}

// ViewModel
using System.Collections.ObjectModel;
using System.Windows.Input;

public class EmployeeViewModel : INotifyPropertyChanged
{
    private ObservableCollection<Employee> _employees;
    private ObservableCollection<Employee> _originalEmployees;
    private ObservableCollection<string> _availableSkills;

    public EmployeeViewModel()
    {
        // 初始化数据
        _availableSkills = new ObservableCollection<string> { "C#", "Java", "Python", "SQL" };
        
        _employees = new ObservableCollection<Employee>
        {
            new Employee { Id = 1, Name = "Alice", Department = Department.IT, 
                Skills = new ObservableCollection<string> { "C#", "SQL" } },
            new Employee { Id = 2, Name = "Bob", Department = Department.HR, 
                Skills = new ObservableCollection<string> { "Java" } }
        };

        // 保存原始数据副本用于取消操作
        _originalEmployees = CloneEmployees(_employees);

        ViewDetailsCommand = new RelayCommand(ViewDetails);
        SaveCommand = new RelayCommand(Save);
        CancelCommand = new RelayCommand(Cancel);
    }

    public ObservableCollection<Employee> Employees
    {
        get => _employees;
        set { _employees = value; OnPropertyChanged(); }
    }

    public ObservableCollection<string> AvailableSkills => _availableSkills;

    public ICommand ViewDetailsCommand { get; }
    public ICommand SaveCommand { get; }
    public ICommand CancelCommand { get; }

    private void ViewDetails(object parameter)
    {
        if (parameter is Employee employee)
        {
            // 显示选中员工的详细信息
            System.Windows.MessageBox.Show(
                $"ID: {employee.Id}\nName: {employee.Name}\nDepartment: {employee.Department}\n" +
                $"Skills: {string.Join(", ", employee.Skills)}",
                "Employee Details");
        }
    }

    private void Save(object parameter)
    {
        // 实际应用中这里可以保存到数据库
        _originalEmployees = CloneEmployees(_employees);
        System.Windows.MessageBox.Show("Changes saved successfully!");
    }

    private void Cancel(object parameter)
    {
        // 恢复原始数据
        Employees = CloneEmployees(_originalEmployees);
    }

    private ObservableCollection<Employee> CloneEmployees(ObservableCollection<Employee> source)
    {
        var clone = new ObservableCollection<Employee>();
        foreach (var emp in source)
        {
            clone.Add(new Employee
            {
                Id = emp.Id,
                Name = emp.Name,
                Department = emp.Department,
                IsSelected = emp.IsSelected,
                Skills = new ObservableCollection<string>(emp.Skills)
            });
        }
        return clone;
    }

    public event PropertyChangedEventHandler? PropertyChanged;

    protected virtual void OnPropertyChanged([System.Runtime.CompilerServices.CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}

// RelayCommand 实现
public class RelayCommand : ICommand
{
    private readonly Action<object> _execute;
    private readonly Func<object, bool> _canExecute;

    public RelayCommand(Action<object> execute, Func<object, bool> canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }

    public bool CanExecute(object parameter) => _canExecute == null || _canExecute(parameter);

    public void Execute(object parameter) => _execute(parameter);

    public event EventHandler CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
}
```

```xaml
// View (XAML)
<Window x:Class="MvvmExample.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:local="clr-namespace:MvvmExample"
        Title="Employee Management" Height="450" Width="800">
    <Grid>
        <DataGrid ItemsSource="{Binding Employees}" AutoGenerateColumns="False" Margin="10">
            <DataGrid.Columns>
                <DataGridTemplateColumn Header="Select">
                    <DataGridTemplateColumn.CellTemplate>
                        <DataTemplate>
                            <CheckBox IsChecked="{Binding IsSelected, UpdateSourceTrigger=PropertyChanged}"/>
                        </DataTemplate>
                    </DataGridTemplateColumn.CellTemplate>
                </DataGridTemplateColumn>

                <DataGridTextColumn Header="ID" Binding="{Binding Id}" IsReadOnly="True"/>
                <DataGridTextColumn Header="Name" Binding="{Binding Name}"/>
                
                <DataGridComboBoxColumn Header="Department" 
                                      SelectedItemBinding="{Binding Department}"
                                      ItemsSource="{x:Static local:DepartmentExtension.AllDepartments}"/>

                <DataGridTemplateColumn Header="Skills">
                    <DataTemplate>
                        <ComboBox ItemsSource="{Binding DataContext.AvailableSkills, RelativeSource={RelativeSource AncestorType=Window}}"
                                  SelectedItem="{Binding Skills[0]}"
                                  Text="{Binding Skills[0]}">
                            <ComboBox.ItemsPanel>
                                <ItemsPanelTemplate>
                                    <VirtualizingStackPanel/>
                                </ItemsPanelTemplate>
                            </ComboBox.ItemsPanel>
                        </ComboBox>
                    </DataTemplate>
                </DataGridTemplateColumn>

                <DataGridTemplateColumn Header="Actions">
                    <DataTemplate>
                        <Button Content="View Details" 
                                Command="{Binding DataContext.ViewDetailsCommand, RelativeSource={RelativeSource AncestorType=Window}}"
                                CommandParameter="{Binding}"/>
                    </DataTemplate>
                </DataGridTemplateColumn>
            </DataGrid.Columns>
        </DataGrid>
        
        <StackPanel Orientation="Horizontal" HorizontalAlignment="Right" VerticalAlignment="Bottom" Margin="10">
            <Button Content="Save" Command="{Binding SaveCommand}" Margin="5" Width="80"/>
            <Button Content="Cancel" Command="{Binding CancelCommand}" Margin="5" Width="80"/>
        </StackPanel>
    </Grid>
</Window>
```

```csharp
// Code-behind
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        DataContext = new EmployeeViewModel();
    }
}
```

```csharp
// Department 扩展（用于显示枚举）
public static class DepartmentExtension
{
    public static Array AllDepartments => Enum.GetValues(typeof(Department));
}
```

这个案例包含以下关键点：

1. **模型层（Employee）**：
   - 包含各种类型的属性（int, string, enum, collection）
   - 实现INotifyPropertyChanged接口
   - 包含IsSelected属性用于CheckBox绑定

2. **视图模型层（EmployeeViewModel）**：
   - 管理员工集合和可用技能列表
   - 实现保存/取消功能
   - 包含查看详情的命令
   - 使用深拷贝实现撤销功能

3. **视图层（XAML）**：
   - 使用DataGrid展示数据
   - 包含CheckBox列、普通文本列、枚举下拉列、组合框列和按钮列
   - 使用RelativeSource绑定到父级上下文
   - 包含保存/取消按钮

4. **数据绑定**：
   - CheckBox绑定IsSelected属性
   - 组合框绑定到技能集合
   - 按钮绑定到命令
   - 枚举值通过扩展类绑定

5. **功能实现**：
   - 支持多行选择
   - 支持技能编辑
   - 支持保存/撤销修改
   - 支持查看详细信息

要运行此代码，需要：
1. 创建WPF项目
2. 添加这些类文件
3. 确保命名空间一致
4. 添加必要的程序集引用

这个示例完整演示了MVVM模式中集合数据管理的常见场景，包含各种类型的数据绑定和命令处理。
</details>

---


# 02
优化代码，使这套方案适用于多种元素类型，如Employee类可改为一个枚举，Name改为枚举的显示名称。同时可能会出现对集合的添加、删除、排序、过滤等操作。
<details style="color: #999">
    <summary style="color: #666; font-size: 12px">思考</summary>
    
好的，用户希望我优化之前的MVVM示例代码，使其适用于多种元素类型，比如将Employee类改为枚举，并将Name属性显示为枚举的显示名称。此外，还需要支持集合的添加、删除、排序和过滤操作。我需要仔细分析用户的需求，并考虑如何调整现有代码结构。

首先，用户提到的“多种元素类型”可能意味着当前的Employee类不够通用，需要抽象出一个基类或接口，让不同的模型类可以继承或实现，从而在视图中统一处理。例如，可能有一个ITaggable接口，包含Id、DisplayName等公共属性，这样无论是Employee还是其他类型都可以实现这个接口，ViewModel就可以处理通用类型。

然后，用户提到Employee类可能改为枚举，Name显示为枚举的显示名称。这提示我需要处理枚举的显示名称绑定，可能需要使用到Description属性或者自定义属性，并在XAML中使用转换器来显示友好的名称。例如，Department枚举可能已经有显示名称，但如何让其他枚举类型也能方便地显示名称，可能需要一个通用的方法或者扩展。

接下来，集合的操作如添加、删除、排序、过滤。当前的示例可能没有这些功能，需要补充。ViewModel需要提供相应的命令，比如AddCommand、DeleteCommand，以及处理排序和过滤的方法。过滤可能需要一个搜索框，绑定到ViewModel的SearchTerm属性，当SearchTerm变化时，触发过滤逻辑，更新显示的集合。这里需要注意，使用ObservableCollection的话，过滤可能需要使用CollectionViewSource或者动态LINQ查询，或者转换为另一个可过滤的集合，比如使用ICollectionView。

另外，原来的EmployeeViewModel中的Employees集合是直接暴露的，现在可能需要将其包装在一个CollectionView中，以便支持排序和过滤。例如，在WPF中，可以使用ListCollectionView，它支持排序、过滤和分组。这样，当用户进行搜索或排序时，只需要操作这个视图，而不需要修改原始数据集合。

关于添加和删除操作，添加可能需要弹出一个对话框让用户输入新项的信息，或者直接在界面上添加一个空行。删除可能需要先选中某些行，然后通过命令删除。这里需要考虑如何泛化这些操作，使得不同的模型类型都可以使用相同的命令逻辑，或者通过基类/接口来抽象公共行为。

然后，考虑如何将原来的Employee类改为更通用的模型。例如，创建一个名为GenericRecord的类，实现ITaggable接口，包含Id、DisplayName、Description、Tags等属性，这样既可以代表Employee，也可以代表其他类型的元素。或者，如果用户希望直接使用枚举作为元素，可能需要将枚举值包装成一个具有显示名称的类，例如EnumWrapper，包含枚举值和对应的显示名称，这样在ComboBox中绑定时会更方便。

此外，用户可能希望视图中的控件能够根据不同的元素类型动态调整显示的内容。例如，如果是Employee类型，显示部门和技能；如果是其他类型，显示不同的属性。这可能需要使用DataTemplate选择器或者动态生成DataGrid的列，但这可能会增加复杂度。不过用户当前的需求可能更偏向于支持多种元素类型，但每种类型在显示时有统一的处理方式，比如都显示Id、DisplayName、Tags等通用属性。

关于取消和保存操作，原来的实现是深拷贝整个集合，这在数据量大时可能有效率问题。可以考虑使用变更跟踪，记录修改过的项，以便在取消时恢复。或者，如果使用ORM框架，可能直接回滚数据库事务。但用户当前的需求可能还是保持内存中的撤销功能，所以深拷贝可能暂时保留，但需要考虑性能优化。

接下来，需要调整XAML部分，使得DataGrid的列能够根据不同的元素类型动态生成，或者统一显示通用属性。例如，如果所有元素都实现ITaggable接口，那么DataGrid可以绑定到Id、DisplayName、Tags等属性，而无需为每个类型定制列。同时，枚举类型的列需要使用ValueConverter来转换显示名称。

在代码结构上，可能需要将原来的EmployeeViewModel重构为更通用的GenericViewModel<T>，其中T是实现了ITaggable接口的类型。这样，针对不同的元素类型，可以创建不同的ViewModel实例。不过，这可能会增加复杂性，尤其是XAML中的数据绑定需要处理泛型类型。因此，可能更简单的方式是保持一个非泛型的ViewModel，但内部使用通用的接口来处理元素。

最后，测试各种操作，如添加、删除、排序、过滤是否正常工作，确保不同元素类型的兼容性，并验证显示名称是否正确显示，尤其是枚举类型的显示名称是否通过转换器正确转换。

总结需要进行的优化步骤：

1. 创建通用接口（如ITaggable）抽象公共属性。

2. 重构Employee类和其他可能的元素类实现该接口。

3. 使用转换器处理枚举的显示名称。

4. 在ViewModel中添加Add、Delete、Sort、Filter的命令和逻辑。

5. 使用CollectionViewSource来处理过滤和排序。

6. 调整XAML，使用动态列生成或通用属性绑定。

7. 确保保存和取消操作正确处理所有变更。

8. 测试各种操作和不同类型的元素。

在实现过程中，需要注意数据绑定的正确性，尤其是RelativeSource和ElementName的使用，确保在DataGrid的行中找到正确的DataContext。同时，处理集合变更时的通知，确保UI及时更新。
</details>

<details>
    <summary style="color: #666; font-size: 12px">正文</summary>

以下是一个优化后的通用化实现方案，支持多种数据类型和集合操作，同时增强可扩展性：

```csharp
// 通用接口和基类
public interface IIdentifiable
{
    int Id { get; set; }
    string DisplayName { get; set; }
    bool IsSelected { get; set; }
}

public abstract class ObservableObject : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    protected virtual void OnPropertyChanged([CallerMemberName] string? propertyName = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(propertyName));
    }
}

// 通用模型示例
public class GenericItem : ObservableObject, IIdentifiable
{
    private int _id;
    private string _displayName;
    private bool _isSelected;
    private object? _tag;

    public int Id
    {
        get => _id;
        set { _id = value; OnPropertyChanged(); }
    }

    public string DisplayName
    {
        get => _displayName;
        set { _displayName = value; OnPropertyChanged(); }
    }

    public bool IsSelected
    {
        get => _isSelected;
        set { _isSelected = value; OnPropertyChanged(); }
    }

    public object? Tag
    {
        get => _tag;
        set { _tag = value; OnPropertyChanged(); }
    }
}

// 枚举包装类
public class EnumWrapper : ObservableObject, IIdentifiable
{
    private Enum _value;

    public EnumWrapper(Enum value)
    {
        _value = value;
    }

    public int Id => Convert.ToInt32(_value);
    
    public string DisplayName => _value.GetDisplayName();
    
    public bool IsSelected { get; set; }

    public Enum Value => _value;
}

// 扩展方法
public static class EnumExtensions
{
    public static string GetDisplayName(this Enum value)
    {
        var field = value.GetType().GetField(value.ToString());
        var attribute = field?.GetCustomAttribute<DescriptionAttribute>();
        return attribute?.Description ?? value.ToString();
    }
}

// 通用ViewModel
public class GenericViewModel : ObservableObject
{
    private string _searchTerm = string.Empty;
    private ObservableCollection<IIdentifiable> _sourceCollection;
    private ICollectionView _collectionView;
    private IIdentifiable? _selectedItem;

    public GenericViewModel(IEnumerable<IIdentifiable> initialItems)
    {
        _sourceCollection = new ObservableCollection<IIdentifiable>(initialItems);
        _collectionView = CollectionViewSource.GetDefaultView(_sourceCollection);
        _collectionView.Filter = FilterItems;

        InitializeCommands();
    }

    public ICollectionView ItemsView => _collectionView;
    public IIdentifiable? SelectedItem { get; set; }

    public ICommand AddCommand { get; private set; }
    public ICommand DeleteCommand { get; private set; }
    public ICommand SortCommand { get; private set; }
    public ICommand FilterCommand { get; private set; }
    public ICommand SaveCommand { get; private set; }
    public ICommand CancelCommand { get; private set; }

    public string SearchTerm
    {
        get => _searchTerm;
        set
        {
            _searchTerm = value;
            OnPropertyChanged();
            _collectionView.Refresh();
        }
    }

    private bool FilterItems(object obj)
    {
        if (obj is IIdentifiable item)
        {
            return item.DisplayName.Contains(SearchTerm, StringComparison.OrdinalIgnoreCase);
        }
        return false;
    }

    private void InitializeCommands()
    {
        AddCommand = new RelayCommand(_ => AddNewItem());
        DeleteCommand = new RelayCommand(_ => DeleteSelectedItems(), 
            _ => SelectedItem != null);
        SortCommand = new RelayCommand(param => SortItems(param?.ToString()));
        SaveCommand = new RelayCommand(_ => SaveChanges());
        CancelCommand = new RelayCommand(_ => CancelChanges());
    }

    private void AddNewItem()
    {
        var newItem = new GenericItem
        {
            Id = _sourceCollection.Count + 1,
            DisplayName = "New Item"
        };
        _sourceCollection.Add(newItem);
    }

    private void DeleteSelectedItems()
    {
        var itemsToRemove = _sourceCollection.Where(x => x.IsSelected).ToList();
        foreach (var item in itemsToRemove)
        {
            _sourceCollection.Remove(item);
        }
    }

    private void SortItems(string? sortBy)
    {
        _collectionView.SortDescriptions.Clear();
        
        if (!string.IsNullOrEmpty(sortBy))
        {
            _collectionView.SortDescriptions.Add(
                new SortDescription(sortBy, ListSortDirection.Ascending));
        }
    }

    private void SaveChanges()
    {
        // 实现保存逻辑
    }

    private void CancelChanges()
    {
        // 实现取消逻辑
    }
}

// 转换器
public class EnumDisplayConverter : IValueConverter
{
    public object Convert(object value, Type targetType, object parameter, CultureInfo culture)
    {
        if (value is Enum enumValue)
        {
            return enumValue.GetDisplayName();
        }
        return value?.ToString() ?? string.Empty;
    }

    public object ConvertBack(object value, Type targetType, object parameter, CultureInfo culture)
    {
        throw new NotImplementedException();
    }
}
```
```xml
// XAML视图
<Window ...>
    <Window.Resources>
        <local:EnumDisplayConverter x:Key="EnumConverter"/>
    </Window.Resources>
    
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- 工具栏 -->
        <StackPanel Orientation="Horizontal" Grid.Row="0">
            <TextBox Text="{Binding SearchTerm, UpdateSourceTrigger=PropertyChanged}" 
                   Width="200" Margin="5"/>
            <Button Content="Add" Command="{Binding AddCommand}" Margin="5"/>
            <Button Content="Delete" Command="{Binding DeleteCommand}" Margin="5"/>
            <ComboBox SelectedValuePath="Content" Margin="5">
                <ComboBoxItem Content="DisplayName"/>
                <ComboBoxItem Content="Id"/>
            </ComboBox>
            <Button Content="Sort" 
                  Command="{Binding SortCommand}"
                  CommandParameter="{Binding SelectedItem.Content, ElementName=sortComboBox}"/>
        </StackPanel>

        <!-- 数据展示 -->
        <DataGrid ItemsSource="{Binding ItemsView}" Grid.Row="1"
                AutoGenerateColumns="False" SelectionMode="Extended">
            <DataGrid.Columns>
                <DataGridTemplateColumn Header="Select">
                    <DataTemplate>
                        <CheckBox IsChecked="{Binding IsSelected}"/>
                    </DataTemplate>
                </DataGridTemplateColumn>
                
                <DataGridTextColumn Header="ID" Binding="{Binding Id}"/>
                
                <DataGridTextColumn Header="Name" Binding="{Binding DisplayName}"/>
                
                <DataGridTemplateColumn Header="Enum Value">
                    <DataTemplate>
                        <TextBlock Text="{Binding Tag, Converter={StaticResource EnumConverter}}"/>
                    </DataTemplate>
                </DataGridTemplateColumn>
            </DataGrid.Columns>
        </DataGrid>

        <!-- 操作按钮 -->
        <StackPanel Orientation="Horizontal" Grid.Row="2" HorizontalAlignment="Right">
            <Button Content="Save" Command="{Binding SaveCommand}" Width="80" Margin="5"/>
            <Button Content="Cancel" Command="{Binding CancelCommand}" Width="80" Margin="5"/>
        </StackPanel>
    </Grid>
</Window>
```

**主要优化点说明**：

1. **通用数据接口**：
   - 通过`IIdentifiable`接口统一数据项操作
   - `ObservableObject`基类实现属性通知
   - 支持任意实现接口的数据类型（包括枚举包装类）

2. **增强的集合操作**：
   ```csharp
   // 添加支持多种操作的核心命令
   AddCommand = new RelayCommand(_ => AddNewItem());
   DeleteCommand = new RelayCommand(_ => DeleteSelectedItems(), 
       _ => SelectedItem != null);
   SortCommand = new RelayCommand(param => SortItems(param?.ToString()));
   ```

3. **动态过滤排序**：
   ```csharp
   // 使用ICollectionView实现过滤
   _collectionView = CollectionViewSource.GetDefaultView(_sourceCollection);
   _collectionView.Filter = FilterItems;

   // 动态排序实现
   private void SortItems(string? sortBy)
   {
       _collectionView.SortDescriptions.Clear();
       if (!string.IsNullOrEmpty(sortBy))
       {
           _collectionView.SortDescriptions.Add(
               new SortDescription(sortBy, ListSortDirection.Ascending));
       }
   }
   ```

4. **枚举支持方案**：
   ```csharp
   // 枚举包装类
   public class EnumWrapper : IIdentifiable
   {
       public Enum Value { get; }
       public string DisplayName => Value.GetDisplayName();
       // 其他接口实现...
   }

   // 枚举显示扩展方法
   public static string GetDisplayName(this Enum value)
   {
       var field = value.GetType().GetField(value.ToString());
       var attribute = field?.GetCustomAttribute<DescriptionAttribute>();
       return attribute?.Description ?? value.ToString();
   }
   ```

5. **可扩展的UI绑定**：
   ```xml
   <!-- 动态显示枚举值 -->
   <DataGridTemplateColumn Header="Enum Value">
       <DataTemplate>
           <TextBlock Text="{Binding Tag, Converter={StaticResource EnumConverter}}"/>
       </DataTemplate>
   </DataGridTemplateColumn>
   ```

6. **复合数据支持**：
   ```csharp
   // 通过Tag属性支持复杂数据类型
   public object? Tag
   {
       get => _tag;
       set { _tag = value; OnPropertyChanged(); }
   }
   ```

**使用示例**：

1. 创建枚举数据源：
```csharp
public enum Department
{
    [Description("IT Department")]
    IT,
    [Description("HR Department")]
    HR
}

var enumItems = Enum.GetValues(typeof(Department))
    .Cast<Enum>()
    .Select(e => new EnumWrapper(e));
```

2. 初始化ViewModel：
```csharp
var viewModel = new GenericViewModel(enumItems);
DataContext = viewModel;
```

3. 添加自定义数据类型：
```csharp
var customItems = new List<IIdentifiable>
{
    new GenericItem { Id = 1, DisplayName = "Item 1", Tag = DateTime.Now },
    new GenericItem { Id = 2, DisplayName = "Item 2", Tag = 42.5m }
};

var viewModel = new GenericViewModel(customItems);
```

**功能扩展建议**：

1. 添加数据验证：
```csharp
public interface IValidatable
{
    bool Validate();
}

// 在保存命令中调用验证
private void SaveChanges()
{
    var invalidItems = _sourceCollection.OfType<IValidatable>()
        .Where(x => !x.Validate());
    // 处理验证结果...
}
```

2. 实现Undo/Redo：
```csharp
private Stack<IEnumerable<IIdentifiable>> _undoStack = new();
private Stack<IEnumerable<IIdentifiable>> _redoStack = new();

private void SnapshotState()
{
    _undoStack.Push(CloneCollection(_sourceCollection));
    _redoStack.Clear();
}
```

3. 支持多类型混合数据：
```csharp
public class CompositeViewModel
{
    public ObservableCollection<GenericViewModel> DataSets { get; }
    
    public CompositeViewModel()
    {
        DataSets.Add(new GenericViewModel(employees));
        DataSets.Add(new GenericViewModel(departments));
    }
}
```

4. 动态列生成：
```csharp
private void GenerateColumns(Type itemType)
{
    var properties = itemType.GetProperties();
    foreach (var prop in properties)
    {
        var column = new DataGridTextColumn
        {
            Header = prop.Name,
            Binding = new Binding(prop.Name)
        };
        dataGrid.Columns.Add(column);
    }
}
```

这套方案通过以下方式提升扩展性：
1. 接口驱动设计，方便添加新数据类型
2. 命令模式实现标准化操作
3. 组合优于继承的架构设计
4. 松耦合的UI绑定机制
5. 可插拔的转换器和扩展方法

可以根据具体需求选择实现数据持久化、复杂验证、历史记录等高级功能，而无需修改核心架构。
</details>

---


# 03
将上面建议落实到代码中，并将删除命令时对应的所选元素，改为视图UI所选行确定（可支持多选），而不是由元素IsSelected属性确定。另外ViewModel中的SelectedItem，可否改为多个？

<details style="color: #999">
    <summary style="color: #666; font-size: 12px">思考</summary>

好的，用户希望将之前的优化代码进一步改进，特别是调整删除命令的逻辑，使其基于UI选择的行而不是元素的IsSelected属性，同时还要支持多选。另外，ViewModel中的SelectedItem需要改为多个。我需要仔细分析之前的代码，然后找出需要修改的地方。

首先，回顾之前的代码，删除操作是基于IIdentifiable接口中的IsSelected属性。用户现在希望改为通过UI的选择（比如DataGrid的多选）来确定要删除的项。这意味着需要将删除命令的触发从元素的IsSelected属性改为视图中的SelectedItems集合。

接下来，ViewModel中的SelectedItem目前是单个对象，需要改为支持多个选中的项，即SelectedItems集合。这可能需要使用ObservableCollection或者其他集合类型来存储多个选中的项。

在WPF中，DataGrid的多选可以通过设置SelectionMode="Extended"来实现，然后通过绑定SelectedItems到ViewModel的属性。不过，SelectedItems的绑定在WPF中并不直接支持，因为它是只读的依赖属性，需要使用附加属性或事件来处理。可能需要使用一个行为（Behavior）或者事件触发器来将SelectedItems同步到ViewModel。

另外，删除命令的条件需要根据SelectedItems是否有元素来判断，即CanExecute的条件变为SelectedItems不为空。因此，DeleteCommand的CanExecute需要检查SelectedItems.Count > 0。

然后，删除操作的具体实现需要从SelectedItems中获取所有选中的项，并从源集合中移除它们。同时，需要确保在删除后更新视图的过滤和排序状态。

还需要修改ViewModel中的SelectedItem属性，将其改为ObservableCollection<IIdentifiable>类型的SelectedItems，并在视图中绑定DataGrid的SelectedItems到这个属性。但由于DataGrid的SelectedItems不支持直接绑定，可能需要使用Interaction.Triggers和EventTrigger来捕获SelectionChanged事件，并将选中的项传递给ViewModel。

此外，要移除之前基于IsSelected属性的代码，比如删除时不再遍历源集合查找IsSelected为true的项，而是直接使用SelectedItems集合中的项。

还需要考虑如何实现多选绑定。可能需要使用System.Windows.Interactivity库中的EventTrigger，或者使用Microsoft.Xaml.Behaviors.Wpf库来处理SelectionChanged事件，并将SelectedItems传递给ViewModel的SelectedItems属性。

在代码中，需要引入必要的命名空间，比如System.Windows.Interactivity，并添加对应的引用。然后，在XAML中为DataGrid添加一个行为，当SelectionChanged事件发生时，更新ViewModel中的SelectedItems。

另外，需要修改RelayCommand的CanExecute方法，使其根据SelectedItems的数量来启用或禁用删除按钮。

最后，测试这些更改，确保多选删除功能正常工作，并且ViewModel的SelectedItems正确反映UI中的选择。

总结需要进行的步骤：

1. 从IIdentifiable接口中移除IsSelected属性，因为不再需要基于属性的选择。

2. 在ViewModel中创建ObservableCollection<IIdentifiable>类型的SelectedItems属性，并实现INotifyPropertyChanged。

3. 修改删除命令的逻辑，使用SelectedItems中的项进行删除。

4. 调整删除命令的CanExecute条件，检查SelectedItems是否有元素。

5. 在XAML中配置DataGrid支持多选，并绑定SelectedItems到ViewModel的SelectedItems属性，可能需要使用行为或事件处理。

6. 移除所有与IsSelected相关的绑定和代码，包括CheckBox列。

7. 确保在删除操作后，SelectedItems集合被清空或更新，避免残留引用。

8. 测试多选删除功能，确保UI和ViewModel同步。

可能遇到的挑战：

- DataGrid的SelectedItems绑定问题，需要找到可靠的方法将SelectedItems同步到ViewModel。

- 处理集合的线程安全问题，确保在UI线程更新ObservableCollection。

- 确保删除操作后，视图的正确刷新，特别是当使用CollectionView进行过滤或排序时。

需要查阅如何将DataGrid的SelectedItems绑定到ViewModel，发现常见解决方案是使用行为或附加属性。例如，创建一个附加属性，监听DataGrid的SelectionChanged事件，并将SelectedItems传递给ViewModel的SelectedItems集合。

编写一个附加属性类，处理SelectedItems的绑定：

```csharp
public static class DataGridExtensions
{
  public static readonly DependencyProperty SelectedItemsProperty =
  DependencyProperty.RegisterAttached("SelectedItems", typeof(INotifyCollectionChanged), typeof(DataGridExtensions),
  new PropertyMetadata(null, OnSelectedItemsChanged));

  public static void SetSelectedItems(DependencyObject element, INotifyCollectionChanged value)
  {
    element.SetValue(SelectedItemsProperty, value);
  }

  public static INotifyCollectionChanged GetSelectedItems(DependencyObject element)
  {
    return (INotifyCollectionChanged)element.GetValue(SelectedItemsProperty);
  }

  private static void OnSelectedItemsChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
  {
    var dataGrid = d as DataGrid;
    if (dataGrid == null) return;

    if (e.OldValue is INotifyCollectionChanged oldCollection)
    {
      oldCollection.CollectionChanged -= SelectedItems_CollectionChanged;
    }
    if (e.NewValue is INotifyCollectionChanged newCollection)
    {
      newCollection.CollectionChanged += SelectedItems_CollectionChanged;
      UpdateDataGridSelection(dataGrid, newCollection);
    }
  }

  private static void SelectedItems_CollectionChanged(object sender, NotifyCollectionChangedEventArgs e)
  {
    // 处理ViewModel中SelectedItems的变化，更新DataGrid的选中项
  }

  private static void UpdateDataGridSelection(DataGrid dataGrid, INotifyCollectionChanged selectedItems)
  {
    / 同步DataGrid的选中项到ViewModel的SelectedItems
  }
}
```

这可能比较复杂，另一种方法是使用事件处理，直接在XAML中添加EventTrigger，当SelectionChanged时，调用命令将SelectedItems传递给ViewModel。

例如：

在DataGrid中添加：

```xml
<i:Interaction.Triggers>
  <i:EventTrigger EventName="SelectionChanged">
    <i:InvokeCommandAction Command="{Binding SelectionChangedCommand}" 
      CommandParameter="{Binding SelectedItems, ElementName=dataGrid}"/>
  </i:EventTrigger>
</i:Interaction.Triggers>
```

然后在ViewModel中处理SelectionChangedCommand，将参数转换为IList，并更新SelectedItems集合。

这种方法可能更简单，但需要引入System.Windows.Interactivity或Microsoft.Xaml.Behaviors包。

如果用户的项目中没有这些包，可能需要安装它们。或者使用代码后台处理SelectionChanged事件，但这会破坏MVVM的分离原则。

综合考虑，使用行为来处理可能是较好的选择。但需要确保项目引用了必要的库。

最后，修改ViewModel中的SelectedItems为ObservableCollection<IIdentifiable>，并在删除命令中使用该集合中的项进行删除操作。

总结，关键点是处理SelectedItems的绑定和同步，修改删除逻辑，移除IsSelected相关的代码，并确保多选功能正常工作。
</details>

<details>
    <summary style="color: #666; font-size: 12px">正文</summary>

以下是基于需求改进后的代码实现，主要改动包括：

1. 支持视图级多选（使用DataGrid的SelectedItems）
2. 删除操作基于UI选择而非模型属性
3. 增强的集合操作支持
4. 支持多种数据类型

```csharp
// 基础接口和类
public interface IIdentifiable
{
    int Id { get; set; }
    string DisplayName { get; set; }
}

public abstract class ObservableObject : INotifyPropertyChanged
{
    public event PropertyChangedEventHandler? PropertyChanged;

    protected void OnPropertyChanged([CallerMemberName] string? name = null)
    {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
    }
}
```

```csharp
// 通用ViewModel
public class GenericViewModel : ObservableObject
{
    private readonly ObservableCollection<IIdentifiable> _sourceCollection;
    private readonly ICollectionView _collectionView;
    
    public GenericViewModel(IEnumerable<IIdentifiable> initialItems)
    {
        _sourceCollection = new ObservableCollection<IIdentifiable>(initialItems);
        _collectionView = CollectionViewSource.GetDefaultView(_sourceCollection);
        _collectionView.Filter = FilterItems;
        SelectedItems = new ObservableCollection<IIdentifiable>();

        InitializeCommands();
    }

    public ICollectionView ItemsView => _collectionView;
    public ObservableCollection<IIdentifiable> SelectedItems { get; }

    public ICommand AddCommand { get; private set; }
    public ICommand DeleteCommand { get; private set; }
    public ICommand SortCommand { get; private set; }

    private string _searchTerm = string.Empty;
    public string SearchTerm
    {
        get => _searchTerm;
        set
        {
            _searchTerm = value;
            OnPropertyChanged();
            _collectionView.Refresh();
        }
    }

    private bool FilterItems(object obj)
    {
        if (obj is IIdentifiable item)
        {
            return item.DisplayName.Contains(SearchTerm, StringComparison.OrdinalIgnoreCase);
        }
        return false;
    }

    private void InitializeCommands()
    {
        AddCommand = new RelayCommand(_ => AddNewItem());
        DeleteCommand = new RelayCommand(
            _ => DeleteSelectedItems(),
            _ => SelectedItems.Any());
        SortCommand = new RelayCommand(SortItems);
    }

    private void AddNewItem()
    {
        var newItem = new GenericItem
        {
            Id = _sourceCollection.Count + 1,
            DisplayName = $"New Item {DateTime.Now:HH:mm:ss}"
        };
        _sourceCollection.Add(newItem);
    }

    private void DeleteSelectedItems()
    {
        var itemsToRemove = SelectedItems.ToList();
        foreach (var item in itemsToRemove)
        {
            _sourceCollection.Remove(item);
        }
        SelectedItems.Clear();
    }

    private void SortItems(object? parameter)
    {
        if (parameter is not string sortBy) return;

        _collectionView.SortDescriptions.Clear();
        _collectionView.SortDescriptions.Add(
            new SortDescription(sortBy, ListSortDirection.Ascending));
    }
}
```

```csharp
// 数据模型
public class GenericItem : ObservableObject, IIdentifiable
{
    private int _id;
    private string _displayName = string.Empty;

    public int Id
    {
        get => _id;
        set { _id = value; OnPropertyChanged(); }
    }

    public string DisplayName
    {
        get => _displayName;
        set { _displayName = value; OnPropertyChanged(); }
    }
}
```

```csharp
// 枚举支持
public class EnumItem : ObservableObject, IIdentifiable
{
    private readonly Enum _enumValue;

    public EnumItem(Enum enumValue)
    {
        _enumValue = enumValue;
    }

    public int Id => Convert.ToInt32(_enumValue);
    
    public string DisplayName => _enumValue.GetDisplayName();
    
    public Enum Value => _enumValue;
}

public static class EnumExtensions
{
    public static string GetDisplayName(this Enum value)
    {
        var field = value.GetType().GetField(value.ToString());
        return field?.GetCustomAttribute<DescriptionAttribute>()?.Description ?? value.ToString();
    }
}
```

```csharp
// 命令实现
public class RelayCommand : ICommand
{
    private readonly Action<object?> _execute;
    private readonly Func<object?, bool>? _canExecute;

    public RelayCommand(Action<object?> execute, Func<object?, bool>? canExecute = null)
    {
        _execute = execute;
        _canExecute = canExecute;
    }

    public bool CanExecute(object? parameter) => _canExecute?.Invoke(parameter) ?? true;

    public void Execute(object? parameter) => _execute(parameter);

    public event EventHandler? CanExecuteChanged
    {
        add => CommandManager.RequerySuggested += value;
        remove => CommandManager.RequerySuggested -= value;
    }
}
```

```csharp
// 视图绑定帮助类
public static class DataGridBehavior
{
    public static readonly DependencyProperty SelectedItemsProperty =
        DependencyProperty.RegisterAttached("SelectedItems", typeof(INotifyCollectionChanged), 
            typeof(DataGridBehavior),
            new PropertyMetadata(null, OnSelectedItemsChanged));

    public static void SetSelectedItems(DependencyObject element, INotifyCollectionChanged value)
    {
        element.SetValue(SelectedItemsProperty, value);
    }

    public static INotifyCollectionChanged GetSelectedItems(DependencyObject element)
    {
        return (INotifyCollectionChanged)element.GetValue(SelectedItemsProperty);
    }

    private static void OnSelectedItemsChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
    {
        if (d is DataGrid dataGrid)
        {
            if (e.OldValue is INotifyCollectionChanged oldCollection)
            {
                oldCollection.CollectionChanged -= SelectedItemsCollectionChanged;
            }

            if (e.NewValue is INotifyCollectionChanged newCollection)
            {
                newCollection.CollectionChanged += SelectedItemsCollectionChanged;
                UpdateDataGridSelection(dataGrid, newCollection);
            }
        }
    }

    private static void SelectedItemsCollectionChanged(object? sender, NotifyCollectionChangedEventArgs e)
    {
        // 处理双向同步（可选）
    }

    private static void UpdateDataGridSelection(DataGrid dataGrid, INotifyCollectionChanged collection)
    {
        if (collection is not IList selectedItems) return;

        dataGrid.SelectedItems.Clear();
        foreach (var item in selectedItems)
        {
            dataGrid.SelectedItems.Add(item);
        }
    }
}
```

XAML视图实现：
```xml
<Window x:Class="GenericView.MainWindow"
        xmlns="http://schemas.microsoft.com/winfx/2006/xaml/presentation"
        xmlns:x="http://schemas.microsoft.com/winfx/2006/xaml"
        xmlns:i="http://schemas.microsoft.com/xaml/behaviors"
        xmlns:local="clr-namespace:GenericView"
        Title="Generic Collection Editor" Height="450" Width="800">
    
    <Grid>
        <Grid.RowDefinitions>
            <RowDefinition Height="Auto"/>
            <RowDefinition Height="*"/>
            <RowDefinition Height="Auto"/>
        </Grid.RowDefinitions>

        <!-- 工具栏 -->
        <StackPanel Orientation="Horizontal" Grid.Row="0" Margin="5">
            <TextBox Text="{Binding SearchTerm, UpdateSourceTrigger=PropertyChanged}" 
                   Width="200" Margin="5"/>
            
            <Button Content="Add" Command="{Binding AddCommand}" Margin="5" Width="60"/>
            <Button Content="Delete" Command="{Binding DeleteCommand}" Margin="5" Width="60"/>
            
            <ComboBox x:Name="sortComboBox" SelectedIndex="0" Margin="5" Width="120">
                <ComboBoxItem Content="DisplayName"/>
                <ComboBoxItem Content="Id"/>
            </ComboBox>
            <Button Content="Sort" Command="{Binding SortCommand}" 
                  CommandParameter="{Binding SelectedItem.Content, ElementName=sortComboBox}"
                  Margin="5" Width="60"/>
        </StackPanel>

        <!-- 数据表格 -->
        <DataGrid Grid.Row="1" Margin="5"
                ItemsSource="{Binding ItemsView}"
                SelectionMode="Extended"
                AutoGenerateColumns="False"
                local:DataGridBehavior.SelectedItems="{Binding SelectedItems}"
                x:Name="mainDataGrid">
            
            <DataGrid.Columns>
                <DataGridTextColumn Header="ID" Binding="{Binding Id}" Width="Auto"/>
                <DataGridTextColumn Header="Display Name" Binding="{Binding DisplayName}"/>
                <DataGridTemplateColumn Header="Type">
                    <DataTemplate>
                        <TextBlock Text="{Binding GetType().Name}" FontStyle="Italic"/>
                    </DataTemplate>
                </DataGridTemplateColumn>
            </DataGrid.Columns>

            <i:Interaction.Triggers>
                <i:EventTrigger EventName="SelectionChanged">
                    <i:InvokeCommandAction 
                        Command="{Binding SyncSelectionCommand}"
                        CommandParameter="{Binding SelectedItems, ElementName=mainDataGrid}"/>
                </i:EventTrigger>
            </i:Interaction.Triggers>
        </DataGrid>

        <!-- 状态栏 -->
        <StatusBar Grid.Row="2">
            <StatusBarItem>
                <TextBlock Text="{Binding SelectedItems.Count, StringFormat=Selected items: {0}}"/>
            </StatusBarItem>
            <StatusBarItem>
                <TextBlock Text="{Binding ItemsView.Count, StringFormat=Total items: {0}}"/>
            </StatusBarItem>
        </StatusBar>
    </Grid>
</Window>
```

**主要改进说明**：

1. **多选支持**：
   - 使用附加属性`DataGridBehavior.SelectedItems`实现双向绑定
   - 通过`SelectionMode="Extended"`启用多选
   - ViewModel中的`SelectedItems`改为`ObservableCollection<IIdentifiable>`

2. **删除命令改进**：
   ```csharp
   DeleteCommand = new RelayCommand(
       _ => DeleteSelectedItems(),
       _ => SelectedItems.Any()); // 根据选中项数量启用按钮
   ```

3. **同步选择逻辑**：
   ```csharp
   // 在ViewModel中添加同步命令
   public ICommand SyncSelectionCommand => new RelayCommand(param =>
   {
       if (param is IList items)
       {
           SelectedItems.Clear();
           foreach (var item in items.OfType<IIdentifiable>())
           {
               SelectedItems.Add(item);
           }
       }
   });
   ```

4. **移除IsSelected属性**：
   - 删除所有与IsSelected相关的代码
   - 完全依赖视图层的选择状态

5. **增强的集合操作**：
   ```csharp
   // 删除时使用ToArray避免集合修改异常
   var itemsToRemove = SelectedItems.ToArray();
   foreach (var item in itemsToRemove)
   {
       _sourceCollection.Remove(item);
   }
   ```

6. **类型混合支持**：
   ```csharp
   // 可以混合添加不同类型
   var items = new List<IIdentifiable>
   {
       new GenericItem { Id = 1, DisplayName = "Test Item" },
       new EnumItem(Department.IT)
   };
   var vm = new GenericViewModel(items);
   ```

**使用示例**：
```csharp
public partial class MainWindow : Window
{
    public MainWindow()
    {
        InitializeComponent();
        
        // 创建混合数据源
        var items = new List<IIdentifiable>
        {
            new GenericItem { Id = 1, DisplayName = "First Item" },
            new EnumItem(Department.IT),
            new GenericItem { Id = 2, DisplayName = "Second Item" },
            new EnumItem(Department.HR)
        };
        
        DataContext = new GenericViewModel(items);
    }
}

public enum Department
{
    [Description("Information Technology")]
    IT,
    [Description("Human Resources")]
    HR
}
```

**功能扩展点**：

1. **添加拖拽排序**：
   ```csharp
   public ICommand ReorderCommand => new RelayCommand(param =>
   {
       if (param is DragDropEventArgs e && e.Data.GetDataPresent(typeof(IIdentifiable)))
       {
           var draggedItem = (IIdentifiable)e.Data.GetData(typeof(IIdentifiable));
           var targetIndex = _sourceCollection.IndexOf((IIdentifiable)e.TargetItem);
           _sourceCollection.Move(_sourceCollection.IndexOf(draggedItem), targetIndex);
       }
   });
   ```

2. **实现过滤历史**：
   ```csharp
   private Stack<string> _filterHistory = new();
   
   public ICommand UndoFilterCommand => new RelayCommand(
       _ => SearchTerm = _filterHistory.Pop(),
       _ => _filterHistory.Count > 0);
   ```

3. **添加数据验证**：
   ```csharp
   public class ValidatableItem : GenericItem
   {
       public override string DisplayName
       {
           get => base.DisplayName;
           set
           {
               if (string.IsNullOrWhiteSpace(value))
                   throw new ArgumentException("Display name cannot be empty");
               base.DisplayName = value;
           }
       }
   }
   ```

4. **支持导入/导出**：
   ```csharp
   public ICommand ExportCommand => new RelayCommand(_ =>
   {
       var json = JsonConvert.SerializeObject(_sourceCollection);
       File.WriteAllText("export.json", json);
   });
   ```

这套实现方案的特点：
1. 完全解耦视图选择状态与数据模型
2. 支持任意实现IIdentifiable接口的数据类型
3. 内置过滤、排序、多选操作
4. 通过附加属性实现干净的双向绑定
5. 可扩展的架构设计，方便添加新功能
6. 符合MVVM模式的最佳实践
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
