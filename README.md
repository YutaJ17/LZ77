# LZ77

## LZ77 - что такое и зачем нужен

LZ77 (Ziv_Lempel, 1977) - алгоритм сжатия без потерь, основанный на идее «повторяющихся кусочков» в данных. Он не кодирует повторы явно, а заменяет повторяющуюся часть ссылкой на встретившуюся недавно такую же последовательность.

Выгодно, если в данных много повторов (логи, тексты, шаблоны). В сочетании с кодированием Хаффмана или арифметическим лежит в основе DEFLATE - алгоритма, который применяется при хранении PNG и в целом как один из методов сжатия архивов.

### Ключевые понятия  
Окно поиска (W) - на сколько символов назад можно ссылаться.  
Буфер предварительного просмотра (L) -  максимальная длина совпадения.  
Типичный токен: (offset, length, next_symbol)  
offset = расстояние назад от текущей позиции до начала совпадения  
length = длина совпадения  
next_symbol = следующий символ после совпадения

### Псевдокод

Принцип: для каждой позиции находим максимально длинное совпадение в окне, если совпадение есть - выдаем (offset, length, next), иначе литерал (0, 0, char).

```cpp
function LZ77_encode(input, W, L):
    pos = 0
    output = []

    while pos < input.length:
        best_offset = 0
        best_length = 0

        window_start = max(0, pos - W)
        // перебираем все возможные начальные позиции совпадения
        for i from window_start to pos - 1:
            length = 0
            // сравниваем символы по одному, пока: они совпадают, не превышают L и не выходят за пределы входа
            while length < L and pos + length < input.length and input[i + length] == input[pos + length]:
                length += 1

            if length > best_length:
                best_length = length
                best_offset = pos - i

        if best_length == 0:
            // нет подходящей ссылки - выводим литерал
            output.append((0, 0, input[pos]))
            pos += 1
        else:
            if pos + best_length < input.length:
                next_symbol = input[pos + best_length]
                output.append((best_offset, best_length, next_symbol))
                pos += best_length + 1
            else:
                // совпадение до конца входа — нет next_symbol
                output.append((best_offset, best_length, EOF))
                pos += best_length

    return output
```

```cpp
function LZ77_decode(tokens):
	output[]
	for each token in tokens:
		(offset, length, next_symbol) // токен
		if offset == 0 and length == 0:
			output.append(next_symbol)
		else:
			start_pos = output.length - offset
			for k from 0 to length - 1:
				output.append(output[start_pos + k])
			if next_symbol != EOF:
				output.append(next_symbol)
	return output
```


Наивная реализация: для каждой позиции сканируем окно - 
O(n * W * l)
Оптимизации:
- Хеш-таблица по префиксу - быстро находит кандидатов начала совпадений.
- Хеш-цепочки: для каждого хеша сохраняется список с таким же префиксом, перебираем только их.
- Бинарные/суффиксные деревья
- Ограничение количества кандидатов для сравнения (например, проверять не все позиции в одной хеш-группе).

## Примеры

### Пример 1

Вход: "ABABABA"
1. pos=0: окно пуст -> (0, 0, 'A') (вывод литерала), pos->1
2. pos=1: окно=['A'] - 'B' не найден -> (0, 0, 'B'), pos->2
3. pos=2: (ищем максмально длинное совпадение: можно сопоставить c i=0 -> последовательность "A B A B")
   best_length=4, offset=2, next='A'
   (символ после совпадения)->токен (2, 4, 'A'), 
   pos->2+4+1=7 -> конец. Токены: (0, 0, 'A'), (0, 0, 'B'), (2, 4, 'A').

### Пример 2

Вход: "a a c a a c a b c a b a a a c"
1. pos=0: окно пуст -> (0, 0, 'a'), pos=1
2. pos=1: находим повтор 'a' в i=0->длина 1, next='c'->(1, 1, 'c'), pos->3
3. pos=3: подстрока начиная 3: "a a c a b..." - лучший match с i=0 дает длину 4, next='b' -> (3, 4, 'b'), pos->8
4. pos=8: подстрока начиная 8: "c a b a a a c" - лучший match с i=5 дает длину 3, next='a' -> (3, 3, 'a'), pos -> 12
5. pos=12: оставшиеся "a a c" - лучший match с i=11 дает длину 2, next='c' -> (1, 2, 'c'), pos->15 (конец)
Полученные токены: (0, 0, 'a'), (1, 1, 'c'), (3, 4, 'b'), (3, 3, 'a'), (1, 2, 'c')
