```java
public class PosToK {
    public static void main(String[] args) throws IOException {
        String posPath = "C:\\Users\\Yeqia\\Downloads\\操作系统.pos";
        transpile(posPath);
    }

    public static void transpile(String posPath) throws IOException {
        String kmPath = posPath.replace(".pos",".km");
        FileInputStream fi = new FileInputStream(posPath);
        int len = fi.available();
        byte[] bytes = new byte[len];
        fi.read(bytes);
        fi.close();
        String posString = new String(bytes, StandardCharsets.UTF_8);

        JSONObject jsonObject = JSONObject.parseObject(posString);
        JSONObject diagram = (JSONObject) jsonObject.get("diagram");
        JSONObject elements = (JSONObject) diagram.get("elements");
        JSONObject result = new JSONObject();
        result.put("root",parseElement(elements));

        FileWriter fw = new FileWriter(kmPath);
        fw.write(result.toJSONString());
        fw.flush();
        fw.close();
        System.out.println("Transpile successfully!");
    }

    public static JSONObject parseElement(JSONObject element){
        String id = element.getString("id");
        String text = element.getString("title");
        String note = element.getString("note");
        JSONArray children = element.getJSONArray("children");
        JSONArray parsedChildren = new JSONArray();
        for (Object child: children){
            JSONObject jo = (JSONObject) child;
            parsedChildren.add(parseElement(jo));
        }
        JSONObject data = new JSONObject();
        data.put("id",id);
        data.put("text",text);
        data.put("note",note);
        JSONObject res = new JSONObject();
        res.put("data",data);
        res.put("children",parsedChildren);
        return res;
    }
}
```

