---
layout: post
title: Openfire 에서 Open Graph 플러그인 구현
---

이 포스팅은 Openfire에서 메시지에 URL을 보내는 경우 URL 페이지의 정보(제목, 내용, 이미지... 등)를 메시지에 전달하는 
기능을 플러그인으로 구현하는 방법을 설명한다.


## Opengraph 플러그인의 주요 기능

- 패킷을 가로채서 메시지 여부 판단
- URL 메시지 여부 판단
- Open Graph 태그 파싱
- 파싱한 태그를 패킷에 Append 해서 브로드캐스팅


- - -

## Opengraph 플러그인 개발
1. 소스를 **[다운로드](https://www.igniterealtime.org/downloads/source.jsp)** 받는다.

2. Eclipse에 소스를 Import 한다.

3. 프로젝트를 생성한다.

4. 메인 클래스를 생성한다.

	```
    public class OpenGraphPlugin implements Plugin, PacketInterceptor  {
    	public OpenGraphPlugin() {
    	}
       
    	public void initializePlugin(PluginManager manager, File pluginDirectory) {
       		Log.info("Open graph parser plugin initialize ...");
    	}
    
    	public void destroyPlugin() {
       		Log.info("Open graph parser plugin destory ...");
    	}
	}
    ```
5. 패킷 가로채기 기능을 추가한다.
	```
    public class OpenGraphPlugin implements Plugin, PacketInterceptor  {
    	private static final Logger Log = LoggerFactory.getLogger(OpenGraphPlugin.class);
    	private InterceptorManager interceptorManager; 
  
    	public OpenGraphPlugin() {
        	interceptorManager = InterceptorManager.getInstance();
    	}
       
    	public void initializePlugin(PluginManager manager, File pluginDirectory) {
       		Log.info("Open graph parser plugin initialize ...");
       
       		// Register a message interceptor manager
       		interceptorManager.addInterceptor(this);       
    	}
        
    	public void destroyPlugin() {
       		Log.info("Open graph parser plugin destory ...");
       
       		// Unregister a message interceptor manager
       		interceptorManager.removeInterceptor(this);
    	}

    	@Override
    	public void interceptPacket(Packet packet, Session session, boolean incoming, boolean processed)
                    throws PacketRejectedException {
        	if(isValidTargetPacket(packet,incoming,processed)) {
            	Packet original = packet;               
                                       
            	if(original instanceof Message) {
                	Message receivedMessage = (Message)original;                           
                	String url = receivedMessage.getBody();
               }      
       		}
    	}
    }
    ```

6. URL 여부를 판단한다.
	```
    @Override
	public void interceptPacket(Packet packet, Session session, boolean incoming, boolean processed)
                    throws PacketRejectedException {
    	if(isValidTargetPacket(packet,incoming,processed)) {
        	Packet original = packet;               
                                      
	        if(original instanceof Message) {
    	        Message receivedMessage = (Message)original;
        	    String url = receivedMessage.getBody();
            
            	if (isUrl(url)) {
                
            	}
        	}            
    	}      
	}
	
    private boolean isUrl(String str) {
        String regex = "[(http(s)?):\\/\\/(www\\.)?a-zA-Z0-9@:%._\\+~#=]{2,256}\\.[a-z]{2,6}\\b([-a-zA-Z0-9@:%_\\+.~#?&//=]*)";
        Pattern p = Pattern.compile(regex);
        Matcher m = p.matcher(str);

        if (m.find()) {
             return true;
        }
       
        return false;
	}
    
    ```

7. Open graph 태그를 파싱한다.
	```
    @Override
	public void interceptPacket(Packet packet, Session session, boolean incoming, boolean processed)
                    throws PacketRejectedException {
    	if(isValidTargetPacket(packet,incoming,processed)) {
        	Packet original = packet;               
                                      
        	if(original instanceof Message) {
            	Message receivedMessage = (Message)original;
            	String url = receivedMessage.getBody();
            
	            if (isUrl(url)) {
    	            if (url.indexOf("http") < 0 && url.indexOf("https") < 0) {
        	            url = "http://" + url;
            	    }
                          
                	OpenGraphTag tag = map.get(url);
        
                	if (tag == null || (MAX_CACHE_DATE < DateUtils.getDiffDate(tag.getCreateDate()))) {
                    	OpenGraphParser parser = new OpenGraphParser();
                    	tag = parser.parser(url);
                    	map.put(url, tag);
                	}                          
            	}
        	}            
    	}      
	}
    ```

	```
    public class OpenGraphParser {
       public OpenGraphParser() {
    }
    public OpenGraphTag parser(String url) {
       OpenGraphTag tag = new OpenGraphTag();
        Map<String, List<String>> result = new HashMap<String,List<String>>();
        String[] metas = new String[]{"og:title", "og:type", "og:image", "og:url", "og:description" };
        try {
            Document doc = Jsoup
                    .connect(url)
                    .get();
            Elements elements = doc.select("meta[property^=og], meta[name^=og]");
            for (Element elm : elements) {
                String target= elm.hasAttr("property") ? "property" : "name";
                if(!result.containsKey(elm.attr(target))){
                    result.put(elm.attr(target), new ArrayList<String>());
                }
                result.get(elm.attr(target)).add(elm.attr("content"));
            }
            for(String meta : metas){
                if (!(result.containsKey(meta) && result.get(meta).size() > 0)){
                    if(meta.equals(metas[0])){
                        result.put(metas[0]
                                , Arrays.asList(new String[]{doc.select("title").eq(0).text()}));
                    } else if (meta.equals(metas[1])){
                        result.put(metas[1]
                                , Arrays.asList(new String[]{"website"}));
                    } else if (meta.equals(metas[2])){
                        result.put(metas[2]
                                , Arrays.asList(new String[]{doc.select("img").eq(0).attr("abs:src")}));
                    } else if (meta.equals(metas[3])){
                        result.put(metas[3]
                                , Arrays.asList(new String[]{doc.baseUri()}));
                    } else if (meta.equals(metas[4])){
                        result.put(metas[4]
                                , Arrays.asList(new String[]{doc.select("meta[property=description], meta[name=description]").eq(0).attr("content")}));
                    }
                }
            }
            for(String meta : result.keySet()) {
                if(meta.equals(metas[0])){
                    tag.setTitle(result.get(meta).get(0));
                } else if (meta.equals(metas[1])){
                    tag.setType(result.get(meta).get(0));
                } else if (meta.equals(metas[2])){
                    tag.setImage(result.get(meta).get(0));
                } else if (meta.equals(metas[3])){
                    tag.setUrl(result.get(meta).get(0));
                } else if (meta.equals(metas[4])){
                    tag.setDescription(result.get(meta).get(0));
                }
            }
            tag.setCreateDate(DateUtils.getCurrentDate());
        } catch (Exception e){
            e.printStackTrace();
        }
        return tag;
    }
}
    ```
8. 패킷에 append 해서 브로드캐스팅 한다.
	```
    @Overridepublic void interceptPacket(Packet packet, Session session, boolean incoming, boolean processed)                    throws PacketRejectedException {    
    	if(isValidTargetPacket(packet,incoming,processed)) {        
        	Packet original = packet;                                                             
            
            if(original instanceof Message) {            
            	Message receivedMessage = (Message)original;            
                String url = receivedMessage.getBody();                        
                
                if (isUrl(url)) {                
                	if (url.indexOf("http") < 0 && url.indexOf("https") < 0) {                    
                    	url = "http://" + url;                
                    }
                    
                    OpenGraphTag tag = map.get(url);         
                    
                    if (tag == null || (MAX_CACHE_DATE < DateUtils.getDiffDate(tag.getCreateDate()))) {            
                    	OpenGraphParser parser = new OpenGraphParser(); 
                        tag = parser.parser(url);   
                        map.put(url, tag);
                    }
                    
                    Element sendFileElement = receivedMessage.addChildElement("x", "jabber:x:og");                							sendFileElement.addElement("title").setText(tag.getTitle());                											sendFileElement.addElement("image").setText(tag.getImage());                											sendFileElement.addElement("url").setText(tag.getUrl());                												sendFileElement.addElement("description").setText(tag.getDescription()); 
                }
            }
        }
    }
    ```


- - -
## Open graph 플러그인 설치
1. Admin Console의 Plugins 메뉴로 이동한다.
2. "파일 선택" 버튼을 클릭해서 Open graph 플러그인을 선택한다.
3. "Upload Plugins" 버튼을 클릭한다.
