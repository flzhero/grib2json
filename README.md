grib2json
=========

一个将 [GRIB2](http://en.wikipedia.org/wiki/GRIB) 文件解码为 JSON 格式的命令行工具。

本工具使用 netCDF-Java GRIB 解码器，该解码器是 University Corporation for Atmospheric Research/Unidata 
开发的 [THREDDS](https://github.com/Unidata/thredds) 项目的一部分。

安装
----

```bash
git clone <this project>
mvn package
```

这将在 target 目录中创建一个 .tar.gz 文件。将该压缩包解压到您选择的位置。

使用方法
--------

`grib2json` 启动脚本位于 `bin` 目录中，需要定义 `JAVA_HOME` 环境变量。

```
> grib2json --help
用法: grib2json [选项] FILE
	[--compact -c] : 启用紧凑的 JSON 格式
	[--data -d] : 打印 GRIB 记录数据
	[--filter.category --fc value] : 选择具有此数值类别的记录
	[--filter.parameter --fp value] : 选择具有此数值参数的记录
	[--filter.surface --fs value] : 选择具有此数值表面类型的记录
	[--filter.value --fv value] : 选择具有此数值表面值的记录
	[--help -h] : 显示此帮助信息
	[--names -n] : 打印数值代码的名称
	[--output -o value] : 将输出写入指定文件（默认为标准输出）
	[--verbose -v] : 启用日志输出到标准输出
```

例如，以下命令将从 GRIB2 文件 _gfs.t18z.pgrbf00.2p5deg.grib2_ 中输出参数 2（U 风分量）、
表面类型 103（地面以上指定高度）、表面值 10.0 米的记录到标准输出。
注意可选的人类可读 _xyzName_ 键和数据数组：

```bash
> grib2json --names --data --fp 2 --fs 103 --fv 10.0 gfs.t18z.pgrbf00.2p5deg.grib2

[
    {
        "header":{
            "discipline":0,
            "disciplineName":"Meteorological products",
            "gribEdition":2,
            "gribLength":27759,
            "center":7,
            "centerName":"US National Weather Service - NCEP(WMC)",
            "parameterNumber":2,
            "parameterNumberName":"U-component_of_wind",
            "parameterUnit":"m.s-1",
            "surface1Type":103,
            "surface1TypeName":"Specified height level above ground",
            "surface1Value":10.0,
            ...
        },
        "data":[
            -2.12,
            -2.27,
            -2.41,
            ...
        ]
    }
]
```

代码修改说明
------------

### 数据顺序修正

本版本对 `GribRecordWriter.java` 进行了修改，以解决 GRIB2 数据扫描模式导致的数据顺序问题：

#### 1. 纬度边界交换
- **修改内容**：交换了 `la1` 和 `la2` 的输出值
- **原因**：GRIB2 文件中的纬度边界可能与实际显示顺序相反
- **影响**：JSON 输出中的 `header.la1` 和 `header.la2` 值已交换

#### 2. 数据数组垂直翻转
- **修改内容**：`data` 数组按行进行垂直翻转
- **原因**：某些 GRIB2 文件使用从下到上的扫描模式（scanMode），导致数据上下颠倒
- **实现方式**：
  - 获取每行的点数（`nx`）和行数（`ny`）
  - 从最后一行开始，逆序输出每一行数据
  - 每行内部的数据顺序保持不变

#### 示例
原始数据网格（3x3）：
```
1 2 3  (第0行)
4 5 6  (第1行)
7 8 9  (第2行)
```

修改前输出：`[7,8,9,4,5,6,1,2,3]`（上下颠倒）

修改后输出：`[1,2,3,4,5,6,7,8,9]`（正确顺序）

这些修改确保了输出的 JSON 数据与地理坐标系统正确对应，适用于气象数据可视化和分析应用。
