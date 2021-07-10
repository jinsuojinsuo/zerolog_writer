## 一、介绍

```

此文件复制于 github.com/rs/zerolog/console.go
原因在于源文件输出的字段不能按顺序输出
所以在此文件修改
此处引入 go get -u github.com/tidwall/gjson

```

## 二、安装

```shell
go get github.com/jinsuojinsuo/zerolog_writer
```

## 三、示例

```go
package main

import (
	"encoding/json"
	"errors"
	"github.com/jinsuojinsuo/zerolog_writer"
	rotatelogs "github.com/lestrrat-go/file-rotatelogs"
	"github.com/rs/zerolog"
	"log"
	"os"
	"time"
)

type databases struct {
}

func (s *databases) Write(p []byte) (n int, err error) {
	//p为json格式的日志
	//fmt.Println("写入到数据库", string(p))
	return len(p), nil
}

func main() {
	log := NewLog()
	for i := 0; i < 2; i++ {
		log := log.With().Int("前缀", i).Logger() //增加日志前缀 每个函数都可以这样执行一下

		m := map[string]interface{}{
			"aaa": 1,
			"bbb": "ccc",
		}
		r, _ := json.Marshal(m)
		err := errors.New("errors0-00")

		//log.Print("hello world", i)     //同fmt.print debug级别
		//log.Printf("hello world %v", i) //格式化输出 debug级别
		//log.Log().Msg("无级别日志")          //无级别日志

		//支持的日志等级
		//log.Panic().Msg("Panic")
		//log.Fatal().Msg("fatal")
		//log.Error().Msg("Error")
		//log.Warn().Msg("Warn")
		//log.Info().Msg("Info")
		//log.Debug().Msg("Debug")
		log.Trace().
			RawJSON("json", r). //原样输出json不会打
			Str("str", "aaaa\naa测试中文aa"). //这里不会解析换行原样输出\n
			Interface("interface", m). //会将m转成json
			Interface("interface2", 1). //转成json输出
			AnErr("err", err). //输出错误不带颜色
			Err(err). //输出错误带颜色
			Msg("Trace 测试换行") //此处可以输出换行
	}

}

func NewLog() zerolog.Logger {
	//创建保存文件每天切换一个文件 目录不存在会自动创建
	logFileName := "./logs/access.log"
	rl, err := rotatelogs.New(
		logFileName+".%Y%m%d",
		rotatelogs.WithLinkName(logFileName), //创建链接文件
	)
	if err != nil {
		log.Fatalln("创建文件失败")
	}
	zerolog.TimeFieldFormat = time.RFC3339Nano //修改时间格式 此为全局修改 好像没办法局部改除非自己写hook
	dbWriter := &databases{}                   //json格式 写入数据库
	consoleWriter := zerolog_writer.ConsoleWriter{ //带颜色
		Out:        os.Stdout,    //输出到控制台
		TimeFormat: time.RFC3339, //时间格式 这里可以与存储中的格式不一样
	}
	multi := zerolog.MultiLevelWriter(consoleWriter, dbWriter, rl) //输出到多个文件
	logger := zerolog.New(multi)
	logger = logger.With().Timestamp().Logger() //输出时间
	logger = logger.With().Caller().Logger()    //输出行号
	logger = logger.Hook(zerolog.HookFunc(func(e *zerolog.Event, level zerolog.Level, message string) { //添加hook
		e.Str("hook", "hook----") //对每条日志增加hook1键
	}))
	return logger
}
```