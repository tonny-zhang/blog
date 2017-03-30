---
title: 把固定格式的 excel 转化成 json
date: 2017-3-30 10:40:11
tags: 
    - blog
    - 工具
    - go
---
## 转化工具产生的背景
服务端和客户端都需要一样的配置文件，但又不想做一个管理后台（平台化简化），同时又得让不懂开发的运维人员方便维护，现用的方案是运维人员维护一个固定格式的 excel 文件，再通过转化工
具把 excel 转成指定的 json 格式供服务端及客户端使用。

项目中需要把指定格式的 excel 文件转化成 json 格式，但之前版本的转化工具个人觉得有很多问题，今天用 `go` 重写了下，给大家分享出来，先不说太多，直接上代码，也可以直接下载[编译完的 windows 平台下的可执行文件](http://download.csdn.net/detail/wodexintiao/9798577)

convert.go 代码如下
```go
package main

import (
	"bufio"
	"encoding/json"
	"fmt"
	"os"
	"regexp"

	"path"

	"io/ioutil"

	"strings"

	"strconv"

	"github.com/tealeg/xlsx"
)

var dirCurrent, err = os.Getwd()

func errPrint(msg string) {
	fmt.Printf("\x1b[31;1m%s\x1b[0m\n", msg)
}
func convert(excelFileName string) {
	excelFileName = strings.Replace(excelFileName, "\\", "/", -1)
	xlFile, err := xlsx.OpenFile(excelFileName)
	if err != nil {
		fmt.Println(err)
	}
	defer func() {
		if r := recover(); r != nil {
			errPrint(fmt.Sprintf("%s 解析错误 panic的内容%v\n", excelFileName, r))
		}
	}()
	for _, sheet := range xlFile.Sheets {
		if strings.Index(sheet.Name, "_") != 0 || len(sheet.Rows) == 0 {
			continue
		}
		rows := sheet.Rows
		nameZH := rows[0].Cells
		types := rows[1].Cells
		nameEN := rows[2].Cells

		lenCell := len(nameZH)
		var lenCellActual = 0
		var headerRow []map[string]string
		for i, cell := range nameZH {
			name, _ := cell.String()
			if len(name) == 0 {
				break
			}
			t, _ := types[i].String()
			en, _ := nameEN[i].String()

			cellHeader := make(map[string]string)
			cellHeader["name"] = name
			cellHeader["type"] = t
			cellHeader["en"] = en
			headerRow = append(headerRow, cellHeader)
			lenCellActual++
		}

		var data []map[string]interface{}
		for _, row := range rows[3:] {
			if len(row.Cells) == lenCell {
				var dMap = make(map[string]interface{})
				for index, cell := range row.Cells {
					if index < lenCellActual {
						d, _ := cell.String()
						en, _ := nameEN[index].String()
						t, _ := types[index].String()
						if t == "int" {
							valNumber, _ := strconv.Atoi(d)
							dMap[en] = valNumber
						} else {
							dMap[en] = d
						}
					}
				}
				data = append(data, dMap)
			}
		}

		result := make(map[string]interface{})
		result["header"] = headerRow
		result["data"] = data
		bResult, _ := json.Marshal(result)

		outputdir := path.Join(dirCurrent, "output")
		os.MkdirAll(outputdir, os.ModePerm)

		regPostfix := regexp.MustCompile("\\..+$")
		filenameNew := regPostfix.ReplaceAllString(path.Base(excelFileName), ".json")
		outfilename := path.Join(outputdir, filenameNew)

		f, err := os.Create(outfilename)
		defer f.Close()
		if err == nil {
			f.Write(bResult)
			fmt.Printf("%s save!\n", outfilename)
		} else {
			fmt.Println(err)
		}
	}
}
func walk(dir string) {
	files, err := ioutil.ReadDir(dir)
	if err == nil {
		for _, file := range files {
			filepath := path.Join(dir, file.Name())
			convert(filepath)
		}
	} else {
		fmt.Println(err)
	}
}
func main() {
	dirExcel := path.Join(dirCurrent, "data")
	if info, err := os.Stat(dirExcel); !os.IsNotExist(err) && info.IsDir() {
		walk(dirExcel)
		reader := bufio.NewReader(os.Stdin)
		fmt.Println("\n\n回车退出...")
		reader.ReadByte()
		os.Exit(0)
	} else {
		errPrint("当前目录下没有用于存放excel文件的data目录")
	}
}
```

## 使用说明
### 项目初始化步骤：
1. 把 `convert.go` 放在项目 src 目录下
1. 运行 `go get github.com/tealeg/xlsx` 安装解析 excel 的第三方依赖包
1. 运行 `go run convert.go` 可直接编译运行，也可以运行 `go build convert.go` 编译成二进制文件直接运行

### 使用方法：
自动行时会在当前可执行文件目录下寻找 `data` 目录，并遍历该目录下所有文件进行解析，并在当前目录下生成 `output` 文件夹，
并把生成的 json 文件同名放入 output 文件夹下。

### excel 格式说明
1. 暂时只支持导出 excel 中 sheet 名为 "_" 为 sheet
1. 第一行为字段中文描述
1. 第二行为字段数据类型
1. 第三行为字段英文名
1. 第四行为空行
1. 从第五行开始为相应数据

#### excel 格式示例
ID | 名称  | 描述
----|------|----
int | string  | string
ID | name  | desc
 |   | 
1 | one  | 第一个
2 | two  | 第二个

#### 转化后的 JSON 示例
```json
{
    "header": [{
        "name": "ID",
        "type": "int",
        "en": "ID"
    }, {
        "name": "名称",
        "type": "string",
        "en": "name"
    }, {
        "name": "描述",
        "type": "string",
        "en": "desc"
    }],
    "data": [{
        "ID": 1,
        "name": "one",
        "desc": "第一个"
    }, {
        "ID": 2,
        "name": "two",
        "desc": "第二个"
    }]
}
```

以上功能还是不很完善，如：没有对文件类型做强验证、没有增加子目录的遍历。。。感兴趣的同学可以参考自行完善代码。

用 `go` 写完此功能后觉得强类型语言在操作 `json` 的时候没有 `nodejs` 方便，感兴趣的同学也可以用 `nodejs` 重写下此功能。