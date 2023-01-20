[![Documentation](https://godoc.org/github.com/recoilme/pudge?status.svg)](https://godoc.org/github.com/recoilme/pudge)
[![Go Report Card](https://goreportcard.com/badge/github.com/recoilme/pudge)](https://goreportcard.com/report/github.com/recoilme/pudge)
[![Build Status](https://travis-ci.org/recoilme/pudge.svg?branch=master)](https://travis-ci.org/recoilme/pudge)
[![Mentioned in Awesome Go](https://awesome.re/mentioned-badge-flat.svg)](https://github.com/avelino/awesome-go)

Table of Contents
=================

* [Description](#description)
* [Usage](#usage)
* [Cookbook](#cookbook)
* [Disadvantages](#disadvantages)
* [Motivation](#motivation)
* [Benchmarks](#benchmarks)
	* [Test 1](#test-1)
	* [Test 4](#test-4)

## Description

Package pudge - это быстрое и простое хранилище ключей и значений, написанное с использованием стандартной библиотеки Go.

В нем представлено следующее:
* Поддержка очень эффективного поиска, вставок и удалений
* Производительность сравнима с хэш-таблицами
* Возможность получать данные в отсортированном порядке, что позволяет выполнять дополнительные операции, такие как сканирование диапазона
* Выберите с помощью limit/offset/from key, с упорядочением или по префиксу
* Безопасен для использования в горутинах
* Экономия пространства
* Очень короткая и простая кодовая база
* Хорошо протестирован, используется в production

![pudge](https://avatars3.githubusercontent.com/u/417177?s=460&v=4)

## Использование


```golang
package main

import (
	"log"

	"github.com/recoilme/pudge"
)

func main() {
	// Close all database on exit
	defer pudge.CloseAll()

	// Set (directories will be created)
	pudge.Set("../test/test", "Hello", "World")

	// Get (lazy open db if needed)
	output := ""
	pudge.Get("../test/test", "Hello", &output)
	log.Println("Output:", output)

	ExampleSelect()
}


//Примеры выбора
func ExampleSelect() {
	cfg := &pudge.Config{
		SyncInterval: 1} // every second fsync
	db, err := pudge.Open("../test/db", cfg)
	if err != nil {
		log.Panic(err)
	}
	defer db.DeleteFile()
	type Point struct {
		X int
		Y int
	}
	for i := 100; i >= 0; i-- {
		p := &Point{X: i, Y: i}
		db.Set(i, p)
	}
	var point Point
	db.Get(8, &point)
	log.Println(point)
	// Output: {8 8}
	// Select 2 keys, from 7 in ascending order
	keys, _ := db.Keys(7, 2, 0, true)
	for _, key := range keys {
		var p Point
		db.Get(key, &p)
		log.Println(p)
	}
	// Output: {8 8}
	// Output: {9 9}
}

```

## Cookbook

 - Храните данные любого типа. Pudge использует кодер / декодер Gob внутри. Никаких ограничений на размер ключей / значений.

```golang
pudge.Set("strings", "Hello", "World")
pudge.Set("numbers", 1, 42)

type User struct {
	Id int
	Name string
}
u := &User{Id: 1, Name: "name"}
pudge.Set("users", u.Id, u)

```
 - Pudge безопасен для использования в горутинах. Вам не нужно создавать / открывать файлы перед использованием. Просто запишите данные в pudge, не беспокойтесь о состоянии. [web server example](https://github.com/recoilme/pixel)

 - Пудж параллелен. Читатели не блокируют читателей, а писатель - блокирует, но из-за отсутствия состояния pudge безопасно использовать несколько файлов для хранения.

 ![Illustration from slowpoke (based on pudge)](https://camo.githubusercontent.com/a1b406485fa8cd52a98d820de706e3fd255941e9/68747470733a2f2f686162726173746f726167652e6f72672f776562742f79702f6f6b2f63332f79706f6b63333377702d70316a63657771346132323164693168752e706e67)


 - Система хранения по умолчанию: как memcache + файловое хранилище. Pudge использует хэш-карту в памяти для ключей и записывает значения в файлы (данные о значениях не хранятся в памяти). Но вы можете использовать режим inmemory для значений с пользовательской конфигурацией:
```golang
cfg = pudge.DefaultConfig()
cfg.StoreMode = 2
db, err := pudge.Open(dbPrefix+"/"+group, cfg)
...
db.Counter(key, val)
```
В этом случае все данные сохраняются в памяти и будут сохранены на диске только при закрытии. 

[Example server for highload, with http api](https://github.com/recoilme/bandit-server)

 - Вы можете использовать pudge в качестве движка для создания баз данных. 
 
 [Example database](https://github.com/recoilme/slowpoke)

 - Не забудьте закрыть все открытые базы данных при завершении работы/ уничтожении.
```golang
 	// Wait for interrupt signal to gracefully shutdown the server 
	quit := make(chan os.Signal)
	signal.Notify(quit, os.Interrupt, os.Kill)
	<-quit
	log.Println("Shutdown Server ...")
	if err := pudge.CloseAll(); err != nil {
		log.Println("Pudge Shutdown err:", err)
	}
 ```
 [example recovery function for gin framework](https://github.com/recoilme/bandit-server/blob/02e6eb9f89913bd68952ec35f6c37fc203d71fc2/bandit-server.go#L89)

 - У Pudge есть примитивный механизм выбора / запроса.
 ```golang
 // Выбрать 2 keys, from 7 в порядке возрастания
	keys, _ := db.Keys(7, 2, 0, true)
// select keys from db where key>7 order by keys asc limit 2 offset 0
 ```

 - Pudge будет хорошо работать на SSD или жестких дисках. Пудж не ест память, или хранилище, или ваш бутерброд. Никаких скрытых задач уплотнения / перебалансировки / изменения размера и так далее. Нет дерева LSM. Нет MMap. Это очень простая база данных с менее чем 500 местоположениями. Это хорошо для [simple social network](https://github.com/recoilme/tgram) или высоконагруженных систем


## Disadvantages

Недостатки

 - Нет системы транзакций. Все операции изолированы, но вы не можете выполнять их пакетно с автоматическим откатом.
 - [Keys](https://godoc.org/github.com/recoilme/pudge#Keys) функция (select/query engine) может быть медленным. Скорость запроса может варьироваться от 10 мс до 1 с на миллион ключей. Pudge не использует BTree / Skiplist или адаптивное дерево координат для хранения ключей в упорядоченном виде при каждой вставке. Операция заказа является "ленивой" и выполняется только при необходимости.
 - Если вам нужно хранилище или база данных для сотен миллионов ключей - взгляните на [Sniper](https://github.com/recoilme/sniper) or [b52](https://github.com/recoilme/b52). Они оптимизированы для высокой нагрузки (pudge - нет).
 - Нет fsync при каждой вставке. Большая часть данных базы данных fsync также выполняется по таймеру
 - Удаленные данные не удаляются физически (но upsert попытается повторно использовать пространство). Прямо сейчас вы можете сжать базу данных только с помощью резервной копии
```golang
pudge.BackupAll("backup")
```
 - Ключи автоматически преобразуются в двоичный код и упорядочиваются с помощью двоичного компаратора. Он прост в использовании, но упорядочение не будет работать корректно, например, для отрицательных чисел
 - Автор проекта не работает в Google или Facebook, и его имя не Говард Чу или Брэд Фитцпатрик. Но я открыт для вопросов или вкладов.


## Motivation

Некоторые базы данных очень хороши для написания. Некоторые базы данных очень хороши для чтения. Но [pudge хорошо сбалансирован для обоих типов операций](https://github.com/recoilme/pogreb-bench). У него есть маленький [симпатичный api](https://godoc.org/github.com/recoilme/pudge), и у них нет скрытых кладбищ. Это просто хэш-карта, где значения записаны в файлах. И вы можете использовать одну базу данных для хранения в памяти / постоянного хранения без состояния без стресса


## Benchmarks

[All tests here](https://github.com/recoilme/pogreb-bench)

***Some tests, MacBook Pro (Retina, 13-inch, Early 2015)***



### Test 1
Number of keys: 1000000
Minimum key size: 16, maximum key size: 64
Minimum value size: 128, maximum value size: 512
Concurrency: 2


|                       | pogreb  | goleveldb | bolt   | badgerdb | pudge  | slowpoke | pudge(mem) |
|-----------------------|---------|-----------|--------|----------|--------|----------|------------|
| 1M (Put+Get), seconds | 187     | 38        | 126    | 34       | 23     | 23       | 2          |
| 1M Put, ops/sec       | 5336    | 34743     | 8054   | 33539    | 47298  | 46789    | 439581     |
| 1M Get, ops/sec       | 1782423 | 98406     | 499871 | 220597   | 499172 | 445783   | 1652069    |
| FileSize,Mb           | 568     | 357       | 552    | 487      | 358    | 358      | 358        |


### Test 4
Number of keys: 10000000
Key size: 8
Value size: 16
Concurrency: 100


|                       | goleveldb | badgerdb | pudge  |
|-----------------------|-----------|----------|--------|
| 10M (Put+Get), seconds| 165       | 120      | 243    |
| 10M Put, ops/sec      | 122933    | 135709   | 43843  |
| 10M Get, ops/sec      | 118722    | 214981   | 666067 |
| FileSize,Mb           | 312       | 1370     | 381    |
