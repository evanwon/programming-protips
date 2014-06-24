# Programming ProTip™: LINQ's *Set* Operations

> Written by [Evan Wondrasek](mailto:evan@evanw.com), June 24, 2014

Among many other features, .NET's [LINQ library](http://msdn.microsoft.com/en-us/library/bb397926.aspx) contains several useful methods for performing "set operations". If you're familiar with databases, you'll recognize these operators which are commonly used to join tables (although the syntax might be a bit different).

> LINQ stands for Language-Integrated Query. It has been available in .NET since .NET Framework 3.5.

In this example, I'll walk you through five basic LINQ set operators, and show some simple examples of how they can be used.

## Preface

In most of these examples, I'll be working with a list of "healthy foods" and a list of "foods which are available in the fridge". To avoid repetition, I'll define those lists here:

``` csharp
var healthyFood = new List<string>()
                      {
                          "Carrots", 
                          "Tofu", 
                          "Lettuce", 
                          "Cucumbers"
                      };
var foodInFridge = new List<string>()
                       {
                           "Cucumbers", 
                           "Cheeseburgers", 
                           "Tofu", 
                           "Pizza", 
                           "Bacon"
                       };
```

## `Union`

The `Union` method returns all unique items from both sets. 

**Set notation:** `{ A, B, C } ∪ { B, C, D } = { A, B, C, D }`.

### Example

In this example, we'll get a list of all healthy foods combined with all food in the fridge, excluding duplicates.

``` csharp
var allHealthyFoodAndAllFoodInFridge = healthyFood.Union(foodInFridge);
```

### Output

```
> Carrots, Tofu, Lettuce, Cucumbers, Cheeseburgers, Pizza, Bacon
```

## `Intersect`

The `Intersect` method returns only the items which occur in both sets ("overlap").

**Set notation:** `{ A, B, C } ∩ { B, C, D } = { B, C }`

### Example

In this example, we'll get a list of only the healthy food in the fridge.

``` csharp
var healthyFoodInFridge = healthyFood.Intersect(foodInFridge);
```

### Output

```
> Tofu, Cucumbers
```

## `Except`

The `Except` method returns the difference between sets (subtraction).

**Set notation:** `{ A, B, C } - { B, C, D } =  { A }`

### Example #1

In this example, we'll get a list of food in our fridge which isn't in the healthy list.

``` csharp
var unhealthyFoodInFridge = foodInFridge.Except(healthyFood)
```

### Output #1

```
> Cheeseburgers, Pizza, Bacon
```

### Example #2

We can also get a list of the healthy food that isn't in the fridge by reversing the order of the sets.

``` csharp
var healthyFoodNotInFridge = healthyFood.Except(foodInFridge).ToList();
```

### Output #2

```
> Carrots, Lettuce
```

## `Concat`

The `Concat` method gets all items from both sets, like `Union`, but includes duplicates (addition).

**Set notation:** `{ A, B, C } + { B, C, D } =  { A, B, C, B, C, D }`

### Example

In this example, we'll get a list of all healthy foods and all food in the fridge, with duplicates.

``` csharp
var concat = healthyFood.Concat(foodInFridge);
```

### Output

```
> Carrots, Tofu, Lettuce, Cucumbers, Cucumbers, Cheeseburgers, Tofu, Pizza, Bacon
```

## `Distinct`

The `Distinct` method allows you to get only the unique items from a single set.

### Example

In this example, we want to remove any duplicate items from our grocery list. (Note that I'm working with a new list in this example.)

``` csharp
var groceryList = new List<string>() { "Milk", "Eggs", "Butter", "Carrots", "Eggs", "Milk" };
var distinctGroceries = groceryList.Distinct();
```

### Output

```
> Milk, Eggs, Butter, Carrots
```

## Additional information

### An important note about LINQ's *Deferred Execution*

LINQ technologies make extensive use of "deferred execution", also known as *lazy loading*. When using LINQ, you're actually creating "promises to query" against your data, and the queries are only executed at the last possible moment. Until then, data is returned as an `IEnumerable`, which can be thought of as a "promise to get this data when you actually need it".

In the examples in this document, each call to methods like `Union()` and `Distinct()` uses deferred execution. This means that although everything is in place to execute the query, the query won't actually be executed until the data is needed. This execution takes place when I observe the data (like with a `foreach` loop), or if I explicitly execute the query by calling a method like `ToList()` or `Count()`.

Deferred execution gives LINQ a tremendous performance advantage, because you can build up complex filters without evaluating the results until they're actually needed. If not understood though, deferred execution can cause unexpected behavior, so I highly recommend studying this feature carefully before using LINQ.

### How are the set operation comparisons made?

In all of the previous examples, I was implicitly using the type-specific default equality comparer when performing set operations. This typically works as expected for simple data types like `strings`, but what if you're working with more complex structures?

LINQ allows you to provide an optional equality comparer argument (a class which implements `IEqualityComparer<T>`). This feature allows you to define your own comparisons.

To explain, let's stop using `strings` to represent food, and switch to a more Object-Oriented method of defining a `Food` class with some meaningful properties. *(Disclaimer: I know nearly nothing about the number of calories in a serving of food.)*

``` csharp
class Food
{
    public string Name { get; private set; }
    public int CaloriesPerServing { get; private set; }

    public Food(string name, int caloriesPerServing)
    {
        Name = name;
        CaloriesPerServing = caloriesPerServing;
    }

    public override string ToString()
    {
        return string.Format("Food: {0}, Calories per Serving: {1}", 
            Name, CaloriesPerServing);
    }
}
```

Now let's make a list of healthy foods, but include three cucumbers, each with slightly different calories per serving.

``` csharp
var carrot = new Food("Carrot", 100);
var celery = new Food("Celery", -10);
var cucumber1 = new Food("Cucumber", 201);
var cucumber2 = new Food("Cucumber", 202);
var cucumber3 = new Food("Cucumber", 203);

var healthyFoods = new List<Food>() { carrot, celery, cucumber1, cucumber2, cucumber3 };
```

If we get the Distinct items in this list using the default comparison system, we see all three of our cucumbers appear, because they're effectively unique items as far as the CLR is concerned.

```
var distinctHealthyFoods = healthyFoods.Distinct();

> Food: Carrot, Calories per Serving: 100
> Food: Celery, Calories per Serving: -10
> Food: Cucumber, Calories per Serving: 201
> Food: Cucumber, Calories per Serving: 202
> Food: Cucumber, Calories per Serving: 203
```

But what if you only want to determine "uniqueness" based simply off the name of the food? Create your own `IEqualityComparer`!

``` csharp
class FoodNameOnlyComparer : IEqualityComparer<Food>
{
    public bool Equals(Food x, Food y)
    {
        // Perform a case-insensitive comparison of the food names
        return x.Name.Equals(y.Name, StringComparison.OrdinalIgnoreCase);
    }

    public int GetHashCode(Food obj)
    {
        // If two things are equal, the should return the same hash code. 
        // Let's use the Name's hash code.
        // (To make this more robust, check for null/empty too)
        return obj.Name.GetHashCode();
    }
}
```

Now we can call `Distinct` on the list again, but this time provide our custom comparer.

```
var distinctHealthyFoods = healthyFoods.Distinct(new FoodNameOnlyComparer());
```

When we run this code, we'll only see **one** Cucumber since our comparison only looks at the `Name` property of the food. Keep in mind that you're no longer aware of the calories per serving of the other cucumbers that you ignored.

```
Food: Carrot, Calories per Serving: 100
Food: Celery, Calories per Serving: -10
Food: Cucumber, Calories per Serving: 201
```

All LINQ set operators I mentioned in this document accept an optional `IEqualityComparer<T>`.

### Further Reading

LINQ Set Operations (MSDN): http://msdn.microsoft.com/en-us/library/bb546153.aspx

`System.Linq` Namespace (MSDN): http://msdn.microsoft.com/en-us/library/vstudio/system.linq%28v=vs.100%29.aspx

`IEqualityComparer<T>` (MSDN): http://msdn.microsoft.com/en-us/library/vstudio/ms132151%28v=vs.100%29.aspx

LINQ and Deferred Execution (MSDN Blogs): http://blogs.msdn.com/b/charlie/archive/2007/12/09/deferred-execution.aspx
