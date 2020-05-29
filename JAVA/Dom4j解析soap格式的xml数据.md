# Dom4j 解析soap格式的xml数据
> 目前只实现了3层结构的xml数据解析。
> 主要对以下格式的数据进行解析处理，实际格式可以根据开发实际需求进行修改关键字的监听。
> 修改监听对象在 getFirstElement()方法中修改值即可。

### 实现以下数据格式的解析
1. 标准格式
2. 嵌套数组
3. 嵌套对象
4. 数组嵌套对象
5. 对象嵌套数组

### 实现类
```java
import com.alibaba.fastjson.JSONObject;
import org.dom4j.*;

import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

public class SoapXmlFormatUtil {

public class SoapXmlFormatUtil {

    public static JSONObject parse(String soapXml){
        JSONObject object = new JSONObject();
        try {
            JSONObject firstElement = getFirstElement(soapXml);
            JSONObject id0 = firstElement.getJSONObject("id0");
            for (String s : id0.keySet()) {
                Object s1 = id0.get(s);
                if (s1 instanceof List){
                    List list = new ArrayList();
                    List<String> stringList = (List<String>) s1;
                    for (String o : stringList) {
                        if (o.contains("#")){
                            list.add(traverseNode(firstElement, o));
                        }else {
                            list.add(o);
                        }
                    }
                    object.put(s, list);
                }else {
                    String key = s1.toString();
                    if (key.contains("#")){
                        object.put(s, traverseNode(firstElement, key));
                    }else {
                        object.put(s, key);
                    }
                }
            }
        } catch (Exception e) {
            e.printStackTrace();
        }
        return object;
    }

    private static JSONObject traverseNode(JSONObject rootElement, String key) throws Exception{
        JSONObject jsonObject = rootElement.getJSONObject(key.replaceAll("#", ""));
        for (String s2 : jsonObject.keySet()) {
            Object o1 = jsonObject.get(s2);
            if (o1 instanceof List){
                List<String> stringList1 = (List<String>) o1;
                List list1 = new ArrayList();
                for (String s3 : stringList1) {
                    if (s3.contains("#")){
                        JSONObject jsonObject1 = rootElement.getJSONObject(s3.replaceAll("#", ""));
                        list1.add(jsonObject1);
                    }else {
                        list1.add(s3);
                    }
                }
                jsonObject.put(s2, list1);
            }else {
                String key3 = o1.toString();
                if (key3.contains("#")){
                    JSONObject jsonObject1 = rootElement.getJSONObject(key3.replaceAll("#", ""));
                    jsonObject.put(s2, jsonObject1);
                }else {
                    jsonObject.put(s2, key3);
                }
            }
        }
        return jsonObject;
    }

    private static JSONObject getFirstElement(String soapXml) throws Exception{
        JSONObject jsonObject = new JSONObject();
        try {
            Document document = DocumentHelper.parseText(soapXml);
            Element rootElement = document.getRootElement();
            Iterator iterator = rootElement.elementIterator();
            while (iterator.hasNext()){
                Element next = (Element) iterator.next();
                Iterator elementIterator = next.elementIterator();
                while (elementIterator.hasNext()){
                    Element next1 = (Element) elementIterator.next();
                    String name = next1.getName();
                    if ("multiRef".equals(name)){
                        String id = next1.attribute("id").getValue();
                        JSONObject multiRefElement = getMultiRefElement(next1);
                        jsonObject.put(id, multiRefElement);
                    }
                }
            }
        } catch (DocumentException e) {
            e.printStackTrace();
        }
        return jsonObject;
    }

    private static JSONObject getMultiRefElement(Element element) throws Exception{
        JSONObject jsonObject = new JSONObject();
        Iterator iterator1 = element.elementIterator();
        while (iterator1.hasNext()){
            Element next = (Element) iterator1.next();
            String text1 = next.getText();
            String name1 = next.getName();
            Attribute attribute = next.attribute("type");
            Attribute attributeHref = next.attribute("href");
            if (attribute != null){
                String attributeValue = attribute.getValue();
                if (attributeValue.toLowerCase().contains("array")){
                    List list = new ArrayList();
                    Iterator iterator2 = next.elementIterator();
                    while (iterator2.hasNext()){
                        Element next2 = (Element) iterator2.next();
                        Attribute attribute1 = next2.attribute("href");
                        if (attribute1 != null){
                            list.add(attribute1.getValue());
                        }else {
                            list.add(next2.getText());
                        }
                    }
                    jsonObject.put(name1, list);
                }else if (attributeValue.toLowerCase().contains("string")){
                    jsonObject.put(name1, text1);
                }
            }else if (attributeHref != null){
                String attributeValueHref = attributeHref.getValue();
                jsonObject.put(name1, attributeValueHref);
            }
        }
        return jsonObject;
    }

}
```

### 实现结果
```java
import com.alibaba.fastjson.JSONObject;

public class Test3 {

    public static void main(String[] args) {
        String message = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" +
                "<soapenv:Envelope xmlns:soapenv=\"http://schemas.xmlsoap.org/soap/envelope/\" xmlns:xsd=\"http://www.w3.org/2001/XMLSchema\" xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\">\n" +
                " <soapenv:Body>\n" +
                "  <ns1:test soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns1=\"http://impl.prbt.cmcc.com\">\n" +
                "   <event href=\"#id0\"/>\n" +
                "  </ns1:test>\n" +
                "  <multiRef id=\"id0\" soapenc:root=\"0\" soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xsi:type=\"ns2:VRBTIssueProductNotifyEvt\" xmlns:soapenc=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns2=\"http://schemas.prbt.cmcc.com\">\n" +
                "   <id0 xsi:type=\"xsd:string\">1</id0>\n" +
                "   <id1 xsi:type=\"xsd:string\">2</id1>\n" +
                "   <id2 xsi:type=\"xsd:string\">3</id2>\n" +
                "   <id3 xsi:type=\"xsd:string\">4</id3>\n" +
                "   <id4 xsi:type=\"xsd:string\">5</id4>\n" +
                "   <id5 xsi:type=\"xsd:string\">6</id5>\n" +
                "  </multiRef>\n" +
                " </soapenv:Body>\n" +
                "</soapenv:Envelope>";

        String message2 = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" +
                "<soapenv:Envelope xmlns:soapenv=\"http://schemas.xmlsoap.org/soap/envelope/\" xmlns:xsd=\"http://www.w3.org/2001/XMLSchema\" xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\">\n" +
                " <soapenv:Body>\n" +
                "  <ns1:test soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns1=\"http://impl.prbt.cmcc.com\">\n" +
                "   <event href=\"#id0\"/>\n" +
                "  </ns1:test>\n" +
                "  <multiRef id=\"id0\" soapenc:root=\"0\" soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xsi:type=\"ns2:VRBTIssueProductNotifyEvt\" xmlns:soapenc=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns2=\"http://schemas.prbt.cmcc.com\">\n" +
                "   <id0 xsi:type=\"xsd:string\">1</id0>\n" +
                "   <id1 xsi:type=\"xsd:string\">2</id1>\n" +
                "   <id2 xsi:type=\"xsd:string\">3</id2>\n" +
                "   <id3 xsi:type=\"xsd:string\">4</id3>\n" +
                "   <id4 xsi:type=\"xsd:string\">5</id4>\n" +
                "   <id5 xsi:type=\"soapenc:Array\" soapenc:arrayType=\"xsd:string[1]\">\n" +
                "       <item>id5</item>\n" +
                "   </id5>\n" +
                "  </multiRef>\n" +
                " </soapenv:Body>\n" +
                "</soapenv:Envelope>";

        String message3 = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" +
                "<soapenv:Envelope xmlns:soapenv=\"http://schemas.xmlsoap.org/soap/envelope/\" xmlns:xsd=\"http://www.w3.org/2001/XMLSchema\" xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\">\n" +
                " <soapenv:Body>\n" +
                "  <ns1:test soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns1=\"http://impl.prbt.cmcc.com\">\n" +
                "   <event href=\"#id0\"/>\n" +
                "  </ns1:test>\n" +
                "  <multiRef id=\"id0\" soapenc:root=\"0\" soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xsi:type=\"ns2:VRBTIssueProductNotifyEvt\" xmlns:soapenc=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns2=\"http://schemas.prbt.cmcc.com\">\n" +
                "   <id0 xsi:type=\"xsd:string\">1</id0>\n" +
                "   <id1 xsi:type=\"xsd:string\">2</id1>\n" +
                "   <id2 xsi:type=\"xsd:string\">3</id2>\n" +
                "   <id3 xsi:type=\"xsd:string\">4</id3>\n" +
                "   <id4 xsi:type=\"xsd:string\">5</id4>\n" +
                "   <id5 href=\"#id1\"/>\n" +
                "  </multiRef>\n" +
                "  <multiRef id=\"id1\" soapenc:root=\"0\" soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xsi:type=\"ns2:VRBTIssueProductNotifyEvt\" xmlns:soapenc=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns2=\"http://schemas.prbt.cmcc.com\">\n" +
                "   <id0 xsi:type=\"xsd:string\">1</id0>\n" +
                "   <id1 xsi:type=\"xsd:string\">2</id1>\n" +
                "   <id2 xsi:type=\"xsd:string\">3</id2>\n" +
                "   <id3 xsi:type=\"xsd:string\">4</id3>\n" +
                "   <id4 xsi:type=\"xsd:string\">5</id4>\n" +
                "   <id5 xsi:type=\"xsd:string\">6</id5>\n" +
                "  </multiRef>\n" +
                " </soapenv:Body>\n" +
                "</soapenv:Envelope>";

        String message4 = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" +
                "<soapenv:Envelope xmlns:soapenv=\"http://schemas.xmlsoap.org/soap/envelope/\" xmlns:xsd=\"http://www.w3.org/2001/XMLSchema\" xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\">\n" +
                " <soapenv:Body>\n" +
                "  <ns1:test soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns1=\"http://impl.prbt.cmcc.com\">\n" +
                "   <event href=\"#id0\"/>\n" +
                "  </ns1:test>\n" +
                "  <multiRef id=\"id0\" soapenc:root=\"0\" soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xsi:type=\"ns2:VRBTIssueProductNotifyEvt\" xmlns:soapenc=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns2=\"http://schemas.prbt.cmcc.com\">\n" +
                "   <id0 xsi:type=\"xsd:string\">1</id0>\n" +
                "   <id1 xsi:type=\"xsd:string\">2</id1>\n" +
                "   <id2 xsi:type=\"xsd:string\">3</id2>\n" +
                "   <id3 xsi:type=\"xsd:string\">4</id3>\n" +
                "   <id4 xsi:type=\"xsd:string\">5</id4>\n" +
                "   <id5 xsi:type=\"soapenc:Array\" soapenc:arrayType=\"xsd:string[1]\">\n" +
                "       <item href=\"#id1\"/>\n" +
                "   </id5>\n" +
                "  </multiRef>\n" +
                "  <multiRef id=\"id1\" soapenc:root=\"0\" soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xsi:type=\"ns2:VRBTIssueProductNotifyEvt\" xmlns:soapenc=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns2=\"http://schemas.prbt.cmcc.com\">\n" +
                "   <id0 xsi:type=\"xsd:string\">1</id0>\n" +
                "   <id1 xsi:type=\"xsd:string\">2</id1>\n" +
                "   <id2 xsi:type=\"xsd:string\">3</id2>\n" +
                "   <id3 xsi:type=\"xsd:string\">4</id3>\n" +
                "   <id4 xsi:type=\"xsd:string\">5</id4>\n" +
                "   <id5 xsi:type=\"xsd:string\">6</id5>\n" +
                "  </multiRef>\n" +
                " </soapenv:Body>\n" +
                "</soapenv:Envelope>";

        String message5 = "<?xml version=\"1.0\" encoding=\"UTF-8\"?>\n" +
                "<soapenv:Envelope xmlns:soapenv=\"http://schemas.xmlsoap.org/soap/envelope/\" xmlns:xsd=\"http://www.w3.org/2001/XMLSchema\" xmlns:xsi=\"http://www.w3.org/2001/XMLSchema-instance\">\n" +
                " <soapenv:Body>\n" +
                "  <ns1:test soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns1=\"http://impl.prbt.cmcc.com\">\n" +
                "   <event href=\"#id0\"/>\n" +
                "  </ns1:test>\n" +
                "  <multiRef id=\"id0\" soapenc:root=\"0\" soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xsi:type=\"ns2:VRBTIssueProductNotifyEvt\" xmlns:soapenc=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns2=\"http://schemas.prbt.cmcc.com\">\n" +
                "   <id0 xsi:type=\"xsd:string\">1</id0>\n" +
                "   <id1 xsi:type=\"xsd:string\">2</id1>\n" +
                "   <id2 xsi:type=\"xsd:string\">3</id2>\n" +
                "   <id3 xsi:type=\"xsd:string\">4</id3>\n" +
                "   <id4 xsi:type=\"xsd:string\">5</id4>\n" +
                "   <id5 href=\"#id1\"/>\n" +
                "  </multiRef>\n" +
                "  <multiRef id=\"id1\" soapenc:root=\"0\" soapenv:encodingStyle=\"http://schemas.xmlsoap.org/soap/encoding/\" xsi:type=\"ns2:VRBTIssueProductNotifyEvt\" xmlns:soapenc=\"http://schemas.xmlsoap.org/soap/encoding/\" xmlns:ns2=\"http://schemas.prbt.cmcc.com\">\n" +
                "   <id0 xsi:type=\"xsd:string\">1</id0>\n" +
                "   <id1 xsi:type=\"xsd:string\">2</id1>\n" +
                "   <id2 xsi:type=\"xsd:string\">3</id2>\n" +
                "   <id3 xsi:type=\"xsd:string\">4</id3>\n" +
                "   <id4 xsi:type=\"xsd:string\">5</id4>\n" +
                "   <id5  xsi:type=\"soapenc:Array\" soapenc:arrayType=\"xsd:string[1]\">\n" +
                "       <item>id5</item>\n" +
                "   </id5>\n" +
                "  </multiRef>\n" +
                " </soapenv:Body>\n" +
                "</soapenv:Envelope>";

        JSONObject parse = SoapXmlFormatUtil.parse(message);
        System.out.println("===========parse============");
        System.out.println(parse);
        JSONObject parse2 = SoapXmlFormatUtil.parse(message2);
        System.out.println("===========parse2============");
        System.out.println(parse2);
        JSONObject parse3 = SoapXmlFormatUtil.parse(message3);
        System.out.println("===========parse3============");
        System.out.println(parse3);
        JSONObject parse4 = SoapXmlFormatUtil.parse(message4);
        System.out.println("===========parse4============");
        System.out.println(parse4);
        JSONObject parse5 = SoapXmlFormatUtil.parse(message5);
        System.out.println("===========parse5============");
        System.out.println(parse5);
    }
}
```
```java
===========parse============
{"id0":"1","id2":"3","id1":"2","id4":"5","id3":"4","id5":"6"}
===========parse2============
{"id0":"1","id2":"3","id1":"2","id4":"5","id3":"4","id5":["id5"]}
===========parse3============
{"id0":"1","id2":"3","id1":"2","id4":"5","id3":"4","id5":{"id0":"1","id2":"3","id1":"2","id4":"5","id3":"4","id5":"6"}}
===========parse4============
{"id0":"1","id2":"3","id1":"2","id4":"5","id3":"4","id5":[{"id0":"1","id2":"3","id1":"2","id4":"5","id3":"4","id5":"6"}]}
===========parse5============
{"id0":"1","id2":"3","id1":"2","id4":"5","id3":"4","id5":{"id0":"1","id2":"3","id1":"2","id4":"5","id3":"4","id5":["id5"]}}
```

### 执行速度
```java
long currentTimeMillis = System.currentTimeMillis();
System.out.println(currentTimeMillis);
for (int i = 0; i < 10000; i++) {
    JSONObject parse = SoapXmlFormatUtil.parse(message);
    JSONObject parse2 = SoapXmlFormatUtil.parse(message2);
    JSONObject parse3 = SoapXmlFormatUtil.parse(message3);
    JSONObject parse4 = SoapXmlFormatUtil.parse(message4);
    JSONObject parse5 = SoapXmlFormatUtil.parse(message5);
}
long timeMillis = System.currentTimeMillis();
System.out.println(timeMillis);
System.out.println((timeMillis-currentTimeMillis)/1000);
```
```java
1590721566666
1590721579405
12
```
> 12秒执行5万条xml数据的解析，平均 4166条/s
