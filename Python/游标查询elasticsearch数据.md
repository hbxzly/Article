```python
#!/usr/bin/env python
# -*- coding: utf-8 -*-
"""
    @Time    : 2019/11/19
    @Author  : LXW
    @Site    : 
    @File    : elasticSearch_utils.py
    @Software: PyCharm
    @Description: 主要用于对es的查询操作（大量数据查询）
"""

from elasticsearch import Elasticsearch


class ElasticSearchUtils:
    def __init__(self, host):
        self.cli = Elasticsearch(hosts=host)

    def search_by_scroll_id(self, index=None, doc_type=None, size=1000, agg=None):
        """
        使用游标的方式滚动查询大量数据
            默认游标过期时间为两分钟
            ElasticSearch
                5.X 版本以下使用 search_type='scan'
                5.X 版本以上使用 sort='_doc'
        :param index: 索引名
        :param doc_type: 文档类型
        :param size: 单次查询请求的数据量
        :param agg: 查询聚合语句
        :return: 全部查询结果
        """
        all_data = []
        hists = self.cli.search(
            index=index,
            doc_type=doc_type,
            scroll='2m',
            sort='_doc',
            size=size,
            body=agg
        )
        scroll_id = hists['_scroll_id']
        scroll_size = hists['hits']['total']
        for hit in hists["hits"]["hits"]:
            all_data.append(hit["_source"])
        # Start scrolling
        while scroll_size > 0:
            page = self.cli.scroll(scroll_id=scroll_id, scroll='2m')
            # Update the scroll ID
            scroll_id = page['_scroll_id']
            # Get the number of results that we returned in the last scroll
            scroll_size = len(page['hits']['hits'])
            for hit in page["hits"]["hits"]:
                all_data.append(hit["_source"])
        return all_data


if __name__ == '__main__':
    body = {
      "size": 1000,
      "query": {
        "bool": {
          "must": [
            {
              "match_all": {}
            },
            {
              "range": {
                "@timestamp": {
                  "gte": 1574149294213,
                  "lte": 1574150194214,
                  "format": "epoch_millis"
                }
              }
            }
          ],
          "must_not": []
        }
      }
    }
    # data = ElasticSearchUtils(["127.0.0.14:9200", "127.0.0.15:9200", "127.0.0.220:9200"]).search_by_scroll_id(index="test-*", doc_type="api", size=1000, body=body)
    data = ElasticSearchUtils("127.0.0.14:9200").search_by_scroll_id(index="test-*", doc_type="test", size=1000, agg=body)
    for h in data:
        print(h)
        break

```
