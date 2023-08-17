# 第十章：批處理

- [Recap](#Recap)


# 不同類型的服務
- 服務（線上系統）
- 批處理系統（離線系統）
- 流處理系統（準實時系統）: 流處理介於線上和離線（批處理）之間，所以有時候被稱為 準實時（near-real-time） 或 準線上（nearline） 處理。
# MapReduce和分散式檔案系統
- Mapper: Mapper 會在每條輸入記錄上呼叫一次，其工作是從輸入記錄中提取鍵值。對於每個輸入，它可以生成任意數量的鍵值對（包括 None）。它不會保留從一個輸入記錄到下一個記錄的任何狀態，因此每個記錄都是獨立處理的。
- Reducer: MapReduce 框架拉取由 Mapper 生成的鍵值對，收集屬於同一個鍵的所有值，並在這組值上迭代呼叫 Reducer。 Reducer 可以產生輸出記錄（例如相同 URL 的出現次數）。
# 處理偏斜
- 單個鍵關聯的大量資料，則 “將具有相同鍵的所有記錄放到相同的位置” 這種模式就被破壞了。例如在社交網路中，大多數使用者可能會與幾百人有連線，但少數名人可能有數百萬的追隨者。這種不成比例的活動資料庫記錄被稱為 關鍵物件（linchpin object）【38】或 熱鍵（hot key）。

# Recap
- 我覺得這邊主要是在講mapReduce的底層實作
- 分散式批處理框架需要解決的兩個主要問題
  - 分割槽
    - 在 MapReduce 中，Mapper 根據輸入檔案塊進行分割槽。Mapper 的輸出被重新分割槽、排序併合併到可配置數量的 Reducer 分割槽中。這一過程的目的是把所有的 相關 資料（例如帶有相同鍵的所有記錄）都放在同一個地方。 
    - 後 MapReduce 時代的資料流引擎若非必要會盡量避免排序，但它們也採取了大致類似的分割槽方法。
- map reduce golang example, 讓我快速了解map reduce的概念，我覺得是跟投票差不多
``` go
/*
計算單字出現率，先用channel做出很多本map, 最後在到Reduce去算出來
*/
package main

import (
	"fmt"
	"strings"
	"sync"
)

func Map(data string, ch chan map[string]int) {
	wordCounts := make(map[string]int)
	for _, word := range strings.Fields(data) {
		wordCounts[word]++
	}
	ch <- wordCounts
}

func Reduce(maps []map[string]int) map[string]int {
	result := make(map[string]int)
	for _, m := range maps {
		for word, count := range m {
			result[word] += count
		}
	}
	return result
}

func main() {
	texts := []string{
		"hello world",
		"goodbye world",
		"hello user",
		"goodbye user",
		"hello hello",
	}

	ch := make(chan map[string]int, len(texts))
	var wg sync.WaitGroup

	for _, text := range texts {
		wg.Add(1)
		go func(data string) {
			defer wg.Done()
			Map(data, ch)
		}(text)
	}

	go func() {
		wg.Wait()
		close(ch)
	}()

	var maps []map[string]int
	for m := range ch {
		maps = append(maps, m)
	}

	result := Reduce(maps)
	fmt.Println(result)
}

```


