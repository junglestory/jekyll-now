---
layout: post
title: Openfire 에서 Open Graph 플러그인 구현
---

페이스북이나 카카오톡에서 URL 링크을 보내면 해당 URL 페이지의 미리보기(제목, 내용, 이미지... 등) 기능을 제공한다.
Openfire에서는 아직 URL 미리보기 기능이 없기에 플러그인으로 구현하는 방법을 올려본다.


## Opengraph 플러그인의 기능 구성

- 패킷을 가로채서 메시지 여부 판단
- URL 메시지 여부 판단
- Open Graph 태그 파싱
- Open Graph 태그가 없는 경우 본문에서 태그 추출
- 파싱한 태그를 패킷에 Append 해서 브로드캐스팅



## Opengraph 플러그인 개발
1. Openfire 소스를 **[다운로드](https://www.igniterealtime.org/downloads/source.jsp)** 받는다.

2. Eclipse에 Openfire 소스를 Import 한다.

3. Opengraph 프로젝트를 생성한다.
	openfire/src/plugins 경로 밑에 opengraph 폴더를 생성한다. 그리고, opengraph 폴더 아래에 src/java/com/mydomain/openfire/opengraph 폴더를 생성하고, plugin.xml 과 pom.xml 파일을 생성한다.
    
    plugin.xml 에는 플러그인의 메인 클래스와 플러그인 정보를 설정한다.
    ```ruby
    <?xml version="1.0" encoding="UTF-8"?>
    <plugin>
        <!-- Main plugin calss -->
        <class>com.mydomain.openfire.opengraph.plugin.OpenGraphPlugin</class>

        <!-- Plugin meta-data -->
        <name>Open Graph Parser</name>
        <description>Parses to Open Graph tags</description>
        <author>mycompany</author>
        <version>1.0.0</version>
        <date>03/20/2018</date>
    </plugin>
    ```
    
    pom.xml 에는 플러그인, 개발자, 라이브러리 그리고 빌드 정보를 설정한다.
    ```
    <project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    	<modelVersion>4.0.0</modelVersion>
    	<parent>
        	<artifactId>plugins</artifactId>
        	<groupId>org.igniterealtime.openfire</groupId>
        	<version>4.2.0</version>
    	</parent>
    	<groupId>org.igniterealtime.openfire.plugins</groupId>
    	<artifactId>openGraph</artifactId>
    	<version>1.0.0</version>
    	<name>Open Graph Plugin</name>
    	<description>Parses to Open Graph tags</description>

    	<developers>
        	<developer>
            	<id>develpoer</id>
            	<name>develpoer</name>
            	<email>developer@mydomain.com</email>
            	<organization>myorganization</organization>
            	<organizationUrl>https://www.mydomain.com</organizationUrl>
        	</developer>
    	</developers>

		<dependencies>  
	    	<dependency>
	      	<groupId>org.jsoup</groupId>
	      	<artifactId>jsoup</artifactId>
	      	<version>1.10.2</version>
	    	</dependency>
	  	</dependencies>
	  
    	<build>
        	<sourceDirectory>src/java</sourceDirectory>
        	<plugins>
            	<plugin>
                	<artifactId>maven-assembly-plugin</artifactId>
            	</plugin>
        	</plugins>
    	</build>
	</project>
    ```    

4.  jsoup 라이브러리를 추가한다.
	[jsoup](https://jsoup.org/download) 라이브러리를 다운로드 받아서, src/java/com/mydomain/openfire/opengraph 폴더에  lib 폴더를 생성 후 다운로드 받은 jsoup 라이브러리를 추가한다.

5. Build Path에 Opengraph Plugin 소스 폴더를 추가한다.
	Eclipse에서 Properties > Java Build Path > Source > Add Folder 버튼을 클릭해서 opengraph > src > java 폴더를 선택한다. 그리고, "OK" 버튼을 클릭해서  Opengraph Plugin 소스 폴더를 Build Path에 추가한다.

6. Plugin 인터페이스를 구현한다.
	src/java/com/mydomain/openfire/opengraph 디렉토리에 OpenGraphPlugin.java 을 생성하고 아래 소스를 입력한다.
	
	```
    import org.jivesoftware.openfire.container.Plugin;
	import org.jivesoftware.openfire.container.PluginManager;
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;
	import java.io.File;

    public class OpenGraphPlugin implements Plugin  {
        private static final Logger Log = LoggerFactory.getLogger(OpenGraphPlugin.class);
        
    	public OpenGraphPlugin() {
    	}
    
    	// 플러그인 초기화
    	public void initializePlugin(PluginManager manager, File pluginDirectory) {
       		Log.info("Open graph parser plugin initialize ...");
    	}
    
    	// 플러그인 종료
    	public void destroyPlugin() {
       		Log.info("Open graph parser plugin destory ...");
    	}
	}
    ```

7. 패킷 가로채기 기능을 추가한다.
	implements 에 PacketInterceptor를 추가한다. 그리고, 생성자에 Instance를 추가하고, initializePlugins에는 메시지 Interceptor를 등록을 추가하고, destoryPlugin에는 등록된 Interceptor를 제거를 추가한다. 메시지 패킷을 확인하기 위해 interceptPacket과 isValidTargetPacket 을 추가한다.
	```
    import org.jivesoftware.openfire.container.Plugin;
	import org.jivesoftware.openfire.container.PluginManager;
	import org.jivesoftware.openfire.interceptor.InterceptorManager;
	import org.jivesoftware.openfire.interceptor.PacketInterceptor;
	import org.jivesoftware.openfire.interceptor.PacketRejectedException;
	import org.slf4j.Logger;
	import org.slf4j.LoggerFactory;
	import java.io.File;
	import org.xmpp.packet.Packet;
	import org.xmpp.packet.Message;
	import org.jivesoftware.openfire.session.Session;

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
	
		private boolean isValidTargetPacket(Packet packet, boolean incoming, boolean processed) {
        	return  !processed && incoming && packet instanceof Message;
		}
	}
    ```

8. URL 여부를 판단한다.
	메시지가 URL인지 확인을 위해 isUrl 메소드를 추가하고, interceptPacket 메소드에 url 확인 코드를 추가한다.

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
		        	
		        }
			}		
		}	
	}
	
    /**
	 * URL 여부 체크
	 * @param str
	 * @return
	 */
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

9. Open graph 태그를 파싱한다.
	Open Graph 태그를 파싱 후 담을 객체를 생성한다.	
	```
    public class OpenGraphTag {
		String url;
    	String image;
    	String type;
    	String siteName;
    	String title;
    	String locale;
    	String description;
    	String createDate;

    	public String getUrl() {
        	return url;
    	}

    	public void setUrl(String url) {
        	this.url = url;
    	}

    	public String getImage() {
        	return image;
    	}

    	public void setImage(String image) {
        	this.image = image;
    	}

    	public String getType() {
        	return type;
    	}

    	public void setType(String type) {
        	this.type = type;
    	}

    	public String getSiteName() {
        	return siteName;
	    }

    	public void setSiteName(String siteName) {
        	this.siteName = siteName;
    	}

    	public String getTitle() {
        	return title;
    	}

    	public void setTitle(String title) {
        	this.title = title;
    	}

    	public String getLocale() {
        	return locale;
    	}

    	public void setLocale(String locale) {
        	this.locale = locale;
    	}

    	public String getDescription() {
        	return description;
    	}

    	public void setDescription(String description) {
        	this.description = description;
    	}

    	public String getCreateDate() {
        	return createDate;
    	}

    	public void setCreateDate(String createDate) {
        	this.createDate = createDate;
    	}
	}
    ```
    
    현재 날짜와 캐시 기간 비교를 위해 날짜 유틸 클래스를 추가한다.
    ```
    import java.text.ParseException;
	import java.text.SimpleDateFormat;
	import java.util.Calendar;
	import java.util.Date;

	public class DateUtils {
		private static String DATE_FORMAT = "yyyy-MM-dd HH:mm:ss";

    	/**
     	* 현재 날짜 구하기
     	* @return
     	*/
    	public static String getCurrentDate() {
        	SimpleDateFormat date = new SimpleDateFormat(DATE_FORMAT);

        	Calendar cal = Calendar.getInstance();
        	String currentDate = date.format(cal.getTime());

        	return currentDate;
    	}

    	/**
     	* 날짜 비교
     	* @param date
     	* @return
     	*/
    	public static long getDiffDate(String date)
    	{
        	long diffDays = 0L;

        	try{
            	SimpleDateFormat format = new SimpleDateFormat( DATE_FORMAT);

            	Date FirstDate = format.parse(date);
            	Date SecondDate = format.parse(getCurrentDate());

            	long diffDate = FirstDate.getTime() - SecondDate.getTime();
            	long diffDateDays = diffDate / ( 24*60*60*1000);

            	diffDays = Math.abs(diffDateDays);
        	} catch(ParseException e) {
	            e.printStackTrace();
     	   }

        	return diffDays;
    	}
	}
    ```
    
    Open Graph 파싱을 위한 클래스를 추가한다.
    ```
    import org.jsoup.Jsoup;
	import org.jsoup.nodes.Document;
	import org.jsoup.nodes.Element;
	import org.jsoup.select.Elements;

	import java.util.ArrayList;
	import java.util.Arrays;
	import java.util.HashMap;
	import java.util.List;
	import java.util.Map;

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

				// Open Graph 태그가 없는 경우 meta 태그와 본문에서 추출
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
    
    interceptPacket 메소드에 Open Graph 파싱 코드를 추가한다.
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
                          
                    // 캐시 저장 확인
                	OpenGraphTag tag = map.get(url);
        
        			/ 캐시에 저장이 안되어 있거나 캐시 저장 기간을 초과한 경우 Open Graph 를 파싱 요청한다.
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
10. 패킷에 append 해서 브로드캐스팅 한다.
	가로채기한 패킷에 "<x xmlns="jabber:x:og"></x>" 엘리멘트를 추가하고, 하위에 "title, image, url, description" 추가한다.
    
	```
    private long MAX_CACHE_DATE = 1L;
    private Map<String, OpenGraphTag> map = new ConcurrentHashMap<String, OpenGraphTag>();
    
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
			        				
                    // 가로채기한 패킷에 Open Graph 정보 추가
					Element sendFileElement = receivedMessage.addChildElement("x", "jabber:x:og");
					sendFileElement.addElement("title").setText(tag.getTitle());
					sendFileElement.addElement("image").setText(tag.getImage());
					sendFileElement.addElement("url").setText(tag.getUrl());
					sendFileElement.addElement("description").setText(tag.getDescription());	
		        }
			}		
		}	
	}
    ```
    
    URL에 Open Graph 정보가 있거나, Meta 정보가 있는 경우에는 클라이언트에 아래와 비슷한 패킷을 받을 것이다. 클라이언트에서는 전달받은 패킷에서 "<x xmlns="jabber:x:og"></x>" 엘리먼트를 파싱해서 UI에 사용하면 된다.
    ```
    <message xmlns="jabber:client" to="user01@localhost/56pmeoc6xd" from="room1@conference.localhost/user01" type="groupchat" id="4">
		<body>https://github.com</body>
		<x xmlns="jabber:x:event">
			<composing/>
		</x>
		<x xmlns="jabber:x:og">
			<title>Build software better, together</title>
			<image>https://assets-cdn.github.com/images/modules/open_graph/github-logo.png</image>
			<url>https://github.com</url>
			<description>GitHub is where people build software. More than 27 million people use GitHub to discover, fork, and contribute to over 80 million projects.</description>
		</x>
		<data xmlns="jabber:x:data" from="user@localhost" roomID="8" stamp="2018-04-20T00:51:22.939Z"/>
	</message>
    ```
    
11. Ant로 빌드한다.
	openfire > build > build.xml 파일을 마우스 오른쪽으로 클릭 후 Run As > External Tools Configurations > Targets 에서 openfire 와 plugins 를 체크한다. 그리고, "Run" 버튼을 클릭해서 빌드를 한다.
	빌드가 성공적으로 끝나면 /Openfire/target/openfire/plugins 폴더에서 opengraphe.jar 파일을 확인 할 수 있다.

전체 소스 코드는 [Github repo](https://github.com/junglestory/openfire-opengraph) 에서 다운로드 받을 수 있다.

## Open graph 플러그인 설치
1. Admin Console의 Plugins 메뉴로 이동한다.
2. "파일 선택" 버튼을 클릭해서 /Openfire/target/openfire/plugins/opengraph.jar 파일을 선택한다.
3. "Upload Plugins" 버튼을 클릭한다.


## 참고자료
- [Openfire Plugin Developer Guide](http://download.igniterealtime.org/openfire/docs/latest/documentation/plugin-dev-guide.html)
- [[XMPP] Openfire 플러그인 개발하기](http://trvoid.blogspot.kr/2013/05/openfire.html)
- [XMPP2APNS](https://github.com/myriky/XMPP2APNS)
