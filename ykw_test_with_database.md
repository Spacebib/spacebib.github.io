# 测试与数据库的交互

传统意义上的单元测试应该避免与数据库交互，因为这样要花更多的精力维护数据库。但是现在各种
框架都提供了很方便的工具让测试数据库驱动的应用变得更加简单。

一般的做法是为测试建立专门的数据库，测试前初始化好需要的数据，然后测试后清空数据库。这样的
做法在数据表少的时候还是可行的。但是随着我们的数据表不断膨胀，测试数量增加，每一次测试运行
都需要花费大量的时间，最终导致我们的 TDD 无法进行。所以我们应该想办法优化。

## Fixtures

一种办法的就是每次只初始化此 test case 需要的数据。cakephp 就支持这种模式。这种模式可以
更精确地控制数据库，避免不必要的 IO 消耗。

## 利用数据库事务回滚

还有一种办法数据库的结构只初始化一次，每个 test case 的数据利用 事务 来完成。这样既不会污染数据，又可以优化测试性能。Laravel 中 RefreshDatabase 就实现了这个功能。
