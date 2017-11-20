# Thinkphp3.2-TranslateApi
thinkphp3.2结合微信公众号和翻译接口实现智能翻译回复
### 基础注意事项：必须要有会微信自动回复基础,该示例是在自动回复功能已经做好的前提下
### 第一步、注册一个百度翻译接口，得到APP_ID和SEC_KEY
### 第二步、在commom\common目录下建一个function.php文件,加上下面的几个函数
```php
function buildSign($query, $appID, $salt, $secKey)//拼接签名sign
{
    $str = $appID . $query . $salt . $secKey;
    $ret = md5($str);
    return $ret;
}
function call($url, $args=null, $method="post", $testflag = 0, $timeout = CURL_TIMEOUT, $headers=array())
{
    $ret = false;
    $i = 0; 
    while($ret === false) 
    {
        if($i > 1)
            break;
        if($i > 0) 
        {
            sleep(1);
        }
        $ret = callOnce($url, $args, $method, false, $timeout, $headers);
        $i++;
    }
    return $ret;
}

function callOnce($url, $args=null, $method="post", $withCookie = false, $timeout = CURL_TIMEOUT, $headers=array())
{
    $ch = curl_init();
    if($method == "post") 
    {
        $data = convert($args);
        curl_setopt($ch, CURLOPT_POSTFIELDS, $data);
        curl_setopt($ch, CURLOPT_POST, 1);
    }
    else 
    {
        $data = convert($args);
        if($data) 
        {
            if(stripos($url, "?") > 0) 
            {
                $url .= "&$data";
            }
            else 
            {
                $url .= "?$data";
            }
        }
    }
    curl_setopt($ch, CURLOPT_URL, $url);
    curl_setopt($ch, CURLOPT_TIMEOUT, $timeout);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, 1);
    if(!empty($headers)) 
    {
        curl_setopt($ch, CURLOPT_HTTPHEADER, $headers);
    }
    if($withCookie)
    {
        curl_setopt($ch, CURLOPT_COOKIEJAR, $_COOKIE);
    }
    $r = curl_exec($ch);
    curl_close($ch);
    return $r;
}

function convert(&$args)
{
    $data = '';
    if (is_array($args))
    {
        foreach ($args as $key=>$val)
        {
            if (is_array($val))
            {
                foreach ($val as $k=>$v)
                {
                    $data .= $key.'['.$k.']='.rawurlencode($v).'&';
                }
            }
            else
            {
                $data .="$key=".rawurlencode($val)."&";
            }
        }
        return trim($data, "&");
    }
    return $args;
}
```
### 第三步、控制器编写接收方法
```php
public function reponseMsg(){
    $postArr = $GLOBALS['HTTP_RAW_POST_DATA']; //获取post数据
    $postObj = simplexml_load_string($postArr);//处理消息类型
    if(strtolower($postObj->MsgType) == 'text'){
    $receive=mb_substr( trim($postObj->Content),0,4,'utf-8');//截取前四个字              
    switch ($receive) {
      case '我要翻译':
          $length=mb_strlen($postObj->Content);
          $postObj->Content=mb_substr($postObj->Content,4,$length-1,'utf-8');
          $APP_ID='你的app_id';
          $SEC_KEY='你的SEC_KEY';
          $CURL_TIMEOUT=10;
          $from='zh';//定义接收中文
          $to='en';//翻译成英文
          $url="http://api.fanyi.baidu.com/api/trans/vip/translate";//发送地址
          $args = array(
              'q' => $postObj->Content,//接受的内容
              'appid' => $APP_ID,
              'salt' => rand(10000,99999),
              'from' => $from,
              'to' => $to,

              );
          $args['sign']= buildSign($postObj->Content, $APP_ID, $args['salt'], $SEC_KEY);//调用function.php的自定义函数拼接、加密
          $res = call($url, $args);
          $contents = json_decode($res, true);
          $content=$contents['trans_result'][0]['dst']; 
      break;
      default:
        $content="若需要翻译服务请回复,'我要翻译+你需要翻译的单词或句子'";
      break;
}
 $template = "<xml>
        <ToUserName><![CDATA[%s]]></ToUserName>
        <FromUserName><![CDATA[%s]]></FromUserName>
        <CreateTime>%s</CreateTime>
        <MsgType><![CDATA[%s]]></MsgType>
        <Content><![CDATA[%s]]></Content>
      </xml>";
      $fromUser = $postObj->ToUserName;
      $toUser   = $postObj->FromUserName;
      $time     = time();
      $msgType  = 'text';
      echo sprintf($template, $toUser, $fromUser, $time, $msgType, $content);
  }
