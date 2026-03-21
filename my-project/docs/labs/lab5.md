# Лабораторная работа №5: Построение бинарного дерева итеративным способом

### Цель работы
Изучение структур данных «дерево» и «очередь», а также реализация итеративного алгоритма построения вложенных словарей, имитирующих структуру бинарного дерева в Python.

### Задание
1. Реализация алгоритма: написать функцию bin_tree_it, которая строит бинарное дерево заданной высоты. 
2. Логика ветвления (Вариант 8): 
   * Корень дерева (root) вводится пользователем. 
   * Левый потомок: $root + root / 2$. 
   * Правый потомок: $root^2$. 
3. Метод построения: использовать итеративный подход с применением очереди (collections.deque) для обхода уровней. 
4. Тестирование: покрыть код Unit-тестами для проверки корректности построения при различных входных данных (отрицательные числа, нулевая высота и т.д.).

### Реализация
Основная логика заключается в использовании очереди для хранения текущих узлов. На каждом уровне мы извлекаем узел, вычисляем значения его потомков и добавляем их обратно в очередь для обработки на следующем шаге.

```python
# Вариант 8. Root = 8; height = 4, left_leaf = root+root/2, right_leaf = root^2
from collections import deque
def main():
    """
        Функция считывает два значения, введенных пользователем.
        __вызов функции bin_tree. Ее параметры:__
            - root - значение корня
            - height - конечная высота
        :return: dict: вложенный словарь со структурой бинарного дерева
        """
    root = int(input('значение root'))
    height = int(input('значение height'))
    tree = bin_tree_it(root, height)
    return tree

def bin_tree_it(root, height, l_b = lambda root: root + root/2, r_b = lambda root: root**2) -> dict:
    """
    Функция осуществляет построение бинарного дерева итеративным способом
    :param root: значение корня дерева
    :param height: значение высоты дерева
    :param l_b: значение левой ветки дерева
    :param r_b: значение правой ветки дерева
    :return: dict: вложенный словарь со структурой бинарного дерева
    """
    """Словарь tree, который хранит всё дерево. Ключ — значение корня дерева, значение — список дочерних узлов"""
    tree = {str(root): []}
    """Очередь, которая хранит текущие значения для вычислений: (словарь текущего узла, значение узла)"""
    q = deque([(tree, root)])
    """Цикл по высоте дерева, каждый раз создает новый уровень"""
    for i in range(height):
        """Перебираем каждый узел текущей высоты"""
        for _ in range(len(q)):
            (nodes, value) = q.popleft()

            """Вычисляем значения веток с помощью функции l_b для левой и r_b для правой"""
            left_value = l_b(value)
            right_value = r_b(value)

            """Создаём левый и правый дочерние узлы как словарь. Ключ — значение узла"""
            left_node = {str(left_value): []}
            right_node = {str(right_value): []}

            """Добавляем к текущему узлу левый и правый дочерние узлы"""
            nodes[str(value)] = [left_node, right_node]
            """Вкладываем новые значения в очередь для вычислений на следующей высоте бинарного дерева"""
            q.append((left_node, left_value))
            q.append((right_node, right_value))
        """Переход к следующему уровню"""
    return tree
```

### Тестирование (Unit Testing)
Для подтверждения корректности работы алгоритма был разработан набор Unit-тестов с использованием библиотеки unittest. Тесты проверяют как стандартные случаи построения дерева, так и граничные условия.

??? info "Показать полный код тестов (test_lab5.py)"
    
    ```python
    import unittest
    from lab5 import bin_tree_it

    class TestBinTreeIterative(unittest.TestCase):
        def test_bin_tree(self):
            actual = bin_tree_it(8, 4)
            expected = {'8': [{'12.0': [{'18.0': [{'27.0': [{'40.5': []}, {'729.0': []}]}, {'324.0': [{'486.0': []}, {'104976.0': []}]}]}, {'144.0': [{'216.0': [{'324.0': []}, {'46656.0': []}]}, {'20736.0': [{'31104.0': []}, {'429981696.0': []}]}]}]}, {'64': [{'96.0': [{'144.0': [{'216.0': []}, {'20736.0': []}]}, {'9216.0': [{'13824.0': []}, {'84934656.0': []}]}]}, {'4096': [{'6144.0': [{'9216.0': []}, {'37748736.0': []}]}, {'16777216': [{'25165824.0': []}, {'281474976710656': []}]}]}]}]}
            self.assertEqual(expected, actual)
    
        def test_bin_tree_height_is_null(self):
            actual = bin_tree_it(8, 0)
            expected = {'8': []}
            self.assertEqual(expected, actual)
    
        def test_bin_tree_height_is_one(self):
            actual = bin_tree_it(8, 1)
            expected = {'8': [{'12.0': []}, {'64': []}]}
            self.assertEqual(expected, actual)
    
        def test_bin_tree_height_is_two(self):
            actual = bin_tree_it(8, 2)
            expected = {'8': [{'12.0': [{'18.0': []}, {'144.0': []}]}, {'64': [{'96.0': []}, {'4096': []}]}]}
            self.assertEqual(expected, actual)
    
        def test_bin_tree_height_is_three(self):
            actual = bin_tree_it(8, 3)
            expected = {'8': [{'12.0': [{'18.0': [{'27.0': []}, {'324.0': []}]}, {'144.0': [{'216.0': []}, {'20736.0': []}]}]}, {'64': [{'96.0': [{'144.0': []}, {'9216.0': []}]}, {'4096': [{'6144.0': []}, {'16777216': []}]}]}]}
            self.assertEqual(expected, actual)
    
        def test_all_null(self):
            actual = bin_tree_it(0,0)
            expected = {'0': []}
            self.assertEqual(expected, actual)
    
        def test_root_is_null(self):
            actual = bin_tree_it(0, 3)
            expected = {'0': [{'0.0': [{'0.0': [{'0.0': []}, {'0.0': []}]}, {'0.0': [{'0.0': []}, {'0.0': []}]}]}, {'0': [{'0.0': [{'0.0': []}, {'0.0': []}]}, {'0': [{'0.0': []}, {'0': []}]}]}]}
            self.assertEqual(expected, actual)
    
        def test_negative_root(self):
            actual = bin_tree_it(-2, 1)
            expected = {'-2': [{'-3.0': []}, {'4': []}]}
            self.assertEqual(expected, actual)
    
        def test_only_two_branches(self):
            actual = bin_tree_it(1, 1)
            expected = {'1': [{'1.5': []}, {'1': []}]}
            self.assertEqual(expected, actual)
    
        def test_symmetry(self):
            actual = bin_tree_it(3, 2)
            left_tree, right_tree = list(actual['3'])
            self.assertEqual(len(left_tree), len(right_tree))
    
        def test_invalid_input_root(self):
            with self.assertRaises(TypeError):
                bin_tree_it('two',  2)
    
        def test_invalid_input_height(self):
            with self.assertRaises(TypeError):
                bin_tree_it(2, 'three')
    
        def test_invalid_inputs(self):
            with self.assertRaises(TypeError):
                bin_tree_it('5', 4.9)
    
        def test_negative_height(self):
            actual = bin_tree_it(5, -1)
            expected = {'5': []}
            self.assertEqual(expected, actual)
    
        def test_inputs_are_none(self):
            with self.assertRaises(TypeError):
                bin_tree_it(None,None)
    
        def test_custom_branch_functions(self):
            actual = bin_tree_it(2, 2, l_b=lambda x: x * 2, r_b=lambda x: x + 3)
            expected = {'2': [{'4': [{'8': []}, {'7': []}]}, {'5': [{'10': []}, {'8': []}]}]}
            self.assertEqual(expected, actual)
    
        def test_custom_branch_functions_with_null_root(self):
            actual = bin_tree_it(0, 2, l_b=lambda x: x+1, r_b=lambda x: x+2)
            expected = {'0': [{'1': [{'2': []}, {'3': []}]}, {'2': [{'3': []}, {'4': []}]}]}
            self.assertEqual(expected, actual)
    if __name__ == '__main__':
    unittest.main()
    ```

### Вывод
Была успешно реализована итеративная модель бинарного дерева. Использование структуры данных deque позволило эффективно организовать построение дерева по уровням. Тесты подтвердили устойчивость алгоритма к различным входным значениям и корректность математических операций при вычислении узлов потомков.
