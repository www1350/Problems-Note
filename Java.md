## POI

### HSSFWorkbook超过**65536**行报错

错误信息 Invalid row number (65536) outside allowable range (0..65535)

**问题原因**  HSSFWorkbook:是操作Excel2003以前（包括2003）的版本，扩展名是.xls，一张表最大支持**65536**行数据，256列，也就是说一个sheet页，最多导出**6w多条数据**。

**解决方法** 

方案1 改用sxssf（注意很耗费堆空间）

XSSFWorkbook:是操作Excel2007-2010的版本，扩展名是.xlsx对于不同版本的EXCEL文档要使用不同的工具类。它的一张表最大支持**1048576**行，16384列。

用法 http://poi.apache.org/spreadsheet/how-to.html#sxssf

方案2 切分sheet

**注意** 
```
SXSSFWorkbook wb = new SXSSFWorkbook(100); // keep 100 rows in memory, exceeding rows will be flushed to disk  

 SXSSFWorkbook wb = new SXSSFWorkbook(-1); // turn off auto-flushing and accumulate all rows in memory     
```
上面第一行限制100行后，每100行会被刷入磁盘，刷入磁盘的将无法使用getRow访问。第二行则会关闭自动刷入，注意可能引起oom



每个sheet都会生成一个临时文件，SXSSF会把数据刷到里面去，临时文件可能变得很大。例如，20MB的csv数据临时xml文件将会超过10亿字节。可以让SXSSF用gzip压缩

```
  SXSSFWorkbook wb = new SXSSFWorkbook(); 
  wb.setCompressTempFiles(true); // temp files will be gzipped
```