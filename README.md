# X Flow Test task

## Example_1 comparison:

Оба примера создают класс игрока со свойством "жизни", где затем вызывается метод удара, который соответственно уменьшает их количество.

**Плюсы и минусы Example_1_1**:

| **Плюсы** | **Минусы** |
| --- | --- |
| Простота, компактность, легкочитаемость | HardCoded значения здоровья и урона |
| Нет зависимостей от сторонних JSON-файлов | Возможно, уместнее добавить метод Hit, а не производить вычисления в вызываемой функции |
|  | Опечатка в "NewPayerHealth" `player = new Player(NewPayerHealth);` |

**Плюсы и минусы Example_1_2:**

| **Плюсы** | **Минусы** |
| --- | --- |
| Гибкость за счет загрузки значений из JSON. Что дает разделение данных и кода. | Нет обработки ошибок для сериализации JSON файлов. (try, catch, etc..) |
| Плюс реализации функции Hit в теле класса Player. Что соответствует принципам SRP |  |
| [Serializable] открывает возможности для сохранения состояния, что удобно для более крупных проектов. |  |

Итог Example 1_2 выглядит более перспективным. Я бы добавил обработчик ошибок и присваивание дефолтных значений в случае ошибок сериализации JSON файлов

## Example_2 comparison:

Оба примера расширяют класс Player до ExtPlayer, где обновляют значения жизней после их изменения.

**Плюсы и минусы Example_2_1**:

| **Плюсы** | **Минусы** |
| --- | --- |
| Событие HealthChanged предоставляет как старое, так и новое значение здоровья, что упрощает логику проверки значительных изменений | Возможно, несколько избыточное использование события для простой проверки здоровья. |

**Плюсы и минусы Example_2_2:**

| **Плюсы** | **Минусы** |
| --- | --- |
| Прямой вызов события innerChanged сразу при добавлении обработчика обеспечивает мгновенное обновление интерфейса при подписке | Не передает старое и новое значения здоровья, что требует добавления previousHealth в ExtProgram |

В обоих примерах я бы поменял бы сравнение
`newHealth - oldHealth < -10` на более читаемое
`oldHealth - newHealth > 10` .

Плюс если связывать ети примеры в Example1_x то нет реализации метотда HitPlayer. Также в **Example_2_1 нет явной реализации обработки** `oldHealth, newHealth`

Я бы предложи л бы использовать события с параметрами

`public event Action<int, int> HealthChanged;`

и добавить обработку этих параметров в класс Player.

## Example_3 comparison:

Оба примера описывают поведение объекта `Player`, который отслеживает врага (`currentEnemy`) и обновляет путь к нему. Если путь невозможен, враг сбрасывается, и движение останавливается.

**Плюсы и минусы Example_3_1**:

| **Плюсы** | **Минусы** |
| --- | --- |
|  Чёткое разделение логики в методах | `UpdatePathToEnemy` тесно связан с внутренним состоянием, что снижает повторное использование |

**Плюсы и минусы Example_3_2:**

| **Плюсы** | **Минусы** |
| --- | --- |
| Использует отдельный метод `TryBuildPathToCoord` для создания пути, улучшая читаемость кода и делает его более универсальным | Логика `Update` более громоздкая |

Я бы объединил все лучшее этих двух вариантов:
- разделение логики как показано в первом примере
- вынес бы в отдельные методы логику по сбросу состоянии и назначения пути

Например:

```csharp
public void Update()
    {
        UpdateMovement();
    }
		
		private void UpdateMovement(){
				if (currentEnemy == null)
		        {
		            ResetMovement();
		            return;
		        }
		
		        var builtPath = TryBuildPathToCoord(currentEnemy.currentPosition);
		        if (builtPath == null)
		        {
		            ResetMovement();
		        }
		        else
		        {
		            SetPath(builtPath);
		        }
     }

    private void ResetMovement()
    {
        currentEnemy = null;
        isMoving = false;
        activeWalkPath = null;
    }

    private void SetPath(List<Vector2> path)
    {
        isMoving = true;
        activeWalkPath = path;
    }

    private List<Vector2> TryBuildPathToCoord(Vector2 target)
    {
        return <много строк кода по созданию пути из currentPosition в target>;
    }

```



## Example_4 comparison:

Оба примера реализуют добавляют панель с читами (обнуление, добавление здоровья и тд…). Отличаются они лишь в разном подходе к регистрации этих читов.

**Плюсы и минусы Example_4_1**:

| **Плюсы** | **Минусы** |
| --- | --- |
| Использование провайдеров `ICheatProvider` делает код более модульным. | Требует больше кода для добавления простых читов. |
| Гибкость добавления новых провайдеров. |  |

**Плюсы и минусы Example_4_2:**

| **Плюсы** | **Минусы** |
| --- | --- |
| Простота реализации | Логика не централизована, что усложняет управление. |

Я бы опять же объединил все лучшее этих двух вариантов:
- вынес бы со второго примера метод создания чита в отдельный метод в `CheatManager`

`var cheat1 = Instantiate(_cheatPrefab, CheatManager.Instance.Panel.transform);cheat1.Setup("Cheat health", () => _health++);`

public class CheatManager
{
public static readonly CheatManager Instance = new CheatManager();

```csharp

public class CheatManager
{
...
		public void AddCheat(string name, Action cheatAction)
		{
		    var cheat = UnityEngine.Object.Instantiate(CheatPrefab, _panel.transform);
		    cheat.Setup(name, cheatAction);
		    _cheats.Add(cheat);
		}
 ...
}

```

## Program.cs refactoring task:

Здесь я сгруппировал повторяющийся код и вынес в отдельные методы для удобства читаемости

```csharp
namespace Cleanup
{
    internal class Program
    {
        private const double TargetChangeTime = 1;

        private double _previousTargetSetTime;
        private bool _isTargetSet;
        private dynamic _lockedCandidateTarget;
        private dynamic _lockedTarget;
        private dynamic _target;
        private dynamic _previousTarget;
        private dynamic _activeTarget;
        private dynamic _targetInRangeContainer;

        public void CleanupTest(Frame frame)
        {
            try
            {
                CleanupInvalidTargets();

                _isTargetSet = false;

                // Sets _activeTarget field
                TrySetActiveTargetFromQuantum(frame);

                // If target exists and can be targeted, it should stay within Target Change Time since last target change
                if (IsTargetValidWithinChangeTime())
                {
                    SetTarget(_target);
                }
                else
                {
                    dynamic[] targets = new dynamic[]
                        { _lockedTarget, _activeTarget, _targetInRangeContainer.GetTarget() };

                    foreach (var target in targets)
                    {
                        if (IsTargetValid(target))
                        {
                            SetTarget(target);
                            break;
                        }
                    }
                }
            }
            finally
            {
                if (_isTargetSet)
                {
                    if (_previousTarget != _target)
                    {
                        _previousTargetSetTime = Time.time;
                    }
                }
                else
                {
                    _target = null;
                }

                TargetableEntity.Selected = _target;
            }
        }

        private void CleanupInvalidTargets()
        {
            if (_lockedCandidateTarget && !_lockedCandidateTarget.CanBeTarget)
            {
                _lockedCandidateTarget = null;
            }

            if (_lockedTarget && !_lockedTarget.CanBeTarget)
            {
                _lockedTarget = null;
            }
        }

        private bool IsTargetValidWithinChangeTime()
        {
            return _target && _target.CanBeTarget && Time.time - _previousTargetSetTime < TargetChangeTime;
        }

        private bool IsTargetValid(dynamic target)
        {
            return target != null && target.CanBeTarget;
        }

        private void SetTarget(dynamic target)
        {
            _target = target;
            _isTargetSet = true;
        }

        // MORE CLASS CODE
    }
}
```

P.S. На все ушло около 1.5 часа. В основном ето больше на верстку и документацию.