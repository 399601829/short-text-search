### 自定制的精准短文本搜索服务

    以公司名称搜索来驱动短文本搜索, 这里做了简化, 实际中会涉及更多的属性, 如公司类型, 所属区域等等, 自定制就有很大的灵活性

### 使用方法

    git clone https://github.com/ysc/short-text-search.git
    cd short-text-search
    unix类操作系统执行：
        chmod +x startup.sh & ./startup.sh
    windows类操作系统执行：
        ./startup.bat
    打开浏览器访问: http://localhost:8080/index.jsp  
    JSON格式的API接口: http://localhost:8080/search_suggest.jsp?kw=%E6%B7%B1%E5%9C%B3%E4%B8%87%E7%A7%91&topN=10&highlight=true
    
### 短文本数据  
    
    短文本数据存放在类路径下的 short_text.txt 文件中
     
    src/main/resources/short_text.txt
    
    可以将该文件替换为自己的数据, 如电影名称 电视节目名称 全国邮政地址等等
    
### 索引格式

    系统启动后, 会在启动目录生成索引文件, 如果格式感兴趣, 可查看这些文件:
    
    ☁  short-text-search [master] ⚡ du -h *.txt                                                                                      [master↑1|✚7…
    2.9M	document.txt
    1.0M	index_id_to_document_id.txt
    124M	invert_index.txt
     33M	invert_index_length_status.txt
    
### 索引维护

    每天03:30全量重建索引, 虽然这里是从文件读, 不会变, 没有重建的必要
    但是实际中, 一般是从数据库或者外部系统读, 如果不需要实时索引, 则有重建的必要
    
    实时索引的用法如下:
    
    // 删除
    SearchService.getShortTextSearcher().deleteIndex(100);
    
    // 更新
    SearchService.getShortTextSearcher().updateIndex(new Document(100, "阿里巴巴"));
    
### 搜索性能分析

    1. 找出评分过程最耗时的前10条查询, 分析命令如下:
    cat  logs/short_text_search_logback* | grep 评分耗时 | awk -F ' - ' '{print $2}' | sort -rn | awk '{print $2,$3,$4}' | head -n 10
    
    结果:
    评分耗时: 141毫秒 228911-1000
    评分耗时: 133毫秒 228914-1000
    评分耗时: 131毫秒 249855-1000
    评分耗时: 129毫秒 249856-1000
    评分耗时: 125毫秒 249860-1000
    评分耗时: 122毫秒 231364-1000
    评分耗时: 114毫秒 249859-1000
    评分耗时: 111毫秒 233657-1000
    评分耗时: 107毫秒 231981-1000
    评分耗时: 107毫秒 231368-1000
    
    第一列是执行时间, 单位为毫秒, 最后一列是搜索序号+搜索最大并发
    
    如果想知道完整查询的时间, 则将 评分 改为 搜索接口总, 分析命令如下:
    cat  logs/short_text_search_logback* | grep 搜索接口总耗时 | awk -F ' - ' '{print $2}' | sort -rn | awk '{print $2,$3,$4}' | head -n 10
    
    结果:
    搜索接口总耗时: 198毫秒 245296-1000
    搜索接口总耗时: 196毫秒 245292-1000
    搜索接口总耗时: 192毫秒 245295-1000
    搜索接口总耗时: 179毫秒 245297-1000
    搜索接口总耗时: 169毫秒 245298-1000
    搜索接口总耗时: 159毫秒 249862-1000
    搜索接口总耗时: 144毫秒 228911-1000
    搜索接口总耗时: 140毫秒 228914-1000
    搜索接口总耗时: 137毫秒 245299-1000
    搜索接口总耗时: 134毫秒 249855-1000
    
    
    
    2. 接下来我们想看看最耗时的这条搜索在其他过程中的耗时情况, 分析命令如下:
    cat  logs/short_text_search_logback* | grep 耗时 | awk -F ' - ' '{print $2}'  | awk '{print $2,$3,$4}' | grep 245296-1000
    
    结果:
    查询解析耗时: 0毫秒 245296-1000
    搜索耗时: 1毫秒 245296-1000
    评分耗时: 20毫秒 245296-1000
    排序耗时: 0毫秒 245296-1000
    搜索接口总耗时: 198毫秒 245296-1000
    
    如果想知道这条查询的详细查询参数, 分析命令如下:
    cat  logs/short_text_search_logback* | grep 245296-1000  | awk -F ' - ' '{print $2}'
    
    结果:
    搜索关键词: beno, topN: 10, highlight: false 245296-1000
    0 查询解析耗时: 0毫秒 245296-1000 
    查询结构: [beno, b, e, n, o, be, en, no, ben, eno] 245296-1000
    1 搜索耗时: 1毫秒 245296-1000
    搜索到的结果文档数: 1283, 总的文档数: 17985, 搜索结果占总文档的比例: 7.1337223 %, 限制后的搜索结果数: 1000, 限制后的搜索结果占总文档的比例: 5.5601892 % 245296-1000 
    20 评分耗时: 20毫秒 245296-1000
    0 排序耗时: 0毫秒 245296-1000
    198 搜索接口总耗时: 198毫秒 245296-1000
    
    
    
    3. 如果想知道大部分查询的评分时间, 分析命令如下:
    cat  logs/short_text_search_logback* | grep 评分耗时 | awk -F ' - ' '{print $2}' | awk -F ' ' '{print $2,$3}' | sort | uniq -c | sort -rn | head -n 10
    
    结果:
    28112 评分耗时: 0毫秒
    14490 评分耗时: 1毫秒
    5513 评分耗时: 2毫秒
    2115 评分耗时: 3毫秒
    1752 评分耗时: 14毫秒
    1197 评分耗时: 13毫秒
    1133 评分耗时: 15毫秒
    892 评分耗时: 4毫秒
    654 评分耗时: 16毫秒
    606 评分耗时: 12毫秒
    567 评分耗时: 5毫秒
    554 评分耗时: 17毫秒
    532 评分耗时: 19毫秒
    528 评分耗时: 18毫秒
    512 评分耗时: 21毫秒
    509 评分耗时: 7毫秒
    492 评分耗时: 22毫秒
    488 评分耗时: 20毫秒
    477 评分耗时: 8毫秒
    472 评分耗时: 23毫秒
    460 评分耗时: 6毫秒
    460 评分耗时: 24毫秒
    450 评分耗时: 25毫秒
    446 评分耗时: 26毫秒
    412 评分耗时: 27毫秒
    380 评分耗时: 9毫秒
    363 评分耗时: 11毫秒
    344 评分耗时: 28毫秒
    329 评分耗时: 10毫秒
    298 评分耗时: 29毫秒
    
    第一列表示评分次数, 说明有28112次评分用的时间都不足1毫秒
    
    4. 分析搜索耗时超过3秒的情况, 分析命令如下:
    cat  logs/short_text_search_logback* | grep 搜索接口总耗时 | awk -F ' - ' '{print $2}' |  awk '{if($1>=3000){print $1,$4,$1}}'| sort -rn | awk '{print $2,$1}' > cost-greater-3s.txt


    5. 分析每一次搜索的耗时情况, 分析命令如下:
    cat  logs/short_text_search_logback* | grep 搜索接口总耗时 | awk -F ' - ' '{print $2}' |  awk '{print $4,$1}' > search_performance.txt

[https://travis-ci.org/ysc/short-text-search](https://travis-ci.org/ysc/short-text-search)
