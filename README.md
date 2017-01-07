# swoole-websocket
demo
 <?php
  4 require ('index.php6');
  5 class server
  6 {
  7    private $serv;
  8 
  9    function __construct(){
 10         $this->serv=new swoole_websocket_server("0.0.0.0",9510);
 11         $this->serv->set([
 12             'work_id' => 2,
 13             'dispatch_mode' => 2,
 14             'daemonize' => 0,
 15             'worker_num' => 4,
 16             'task_worker_num' => 4,
 17             'reactor_num' =>4
 18         ]);
 19 
 20         $this->serv->on('start', array($this, 'onStart'));
 21         $this->serv->on('open', array($this, 'update'));
 22         $this->serv->on('message', array($this, 'onMessage'));
 23         $this->serv->on('task', array($this, 'onTask'));
 24         $this->serv->on('finish', array($this, 'onFinish'));
 25 
 26         $port1=$this->serv->listen("0.0.0.0",9506,SWOOLE_SOCK_TCP);
 27         $port1->set([
 28              'open_eof_split' => true,
 29              'package_eof' => "#*"
 30         ]);
 
        $port1->on('Receive', array($this, 'onTCPReceive'));
 33         $port1->on('connect', array($this, 'onConnect'));
 34 
 35         $this->serv->start();
 36    }
 37    public function onStart(){
 38         echo "Start";
 39    }
 40 
 41    public function onConnect($serv,$fd,$from_id){
 42         echo "{$fd}\n";
 43         $redis=new Async\RedisClient('127.0.0.1');
 44         $redis->set('hy',"{$fd}",function() use($redis){
 45         echo "redis is OK:\n";
 46         });
 47         echo "Redis is Already";
 48    }
 49 
 50    public function update(swoole_websocket_server $serv, $request){
 51 
 52          echo "server: handshake success with fd{$request->fd}\n";
 53    }
 54 
 55    public function onMessage(swoole_websocket_server $serv, $frame){
 56 
 57 //      echo "receive from {$frame->fd}:{$frame->data},opcode:{$frame->opcode},fin:{$frame->finish}\n";
 58 //      echo "{$frame->data}";
 59         $data=[
 60              'task'=> 'task_1',
  61              'params'=> $frame->data,
 62              'fd' => $frame->fd
 63         ];
 64         $serv->task( json_encode($data,true));
 65    }
 66 
 67 
 68    public function onTask(swoole_websocket_server $serv, $task_id, $from_id, $data){
 69         $redis2=new Redis();
 70         $redis2->connect("127.0.0.1",6379);
 71         $ffd=$redis2->get('hy');
 72         echo "redis accept is {$ffd}";
 73 //      echo "task id:{$task_id} from id:{$from_id}\n";
 74 //      echo "data :{$data}\n";
 75         $data = json_decode($data,true);
 76 //      var_dump($data);
 77 //      echo "Receive task {$data['task']}\n";
 78 //      var_dump($data['fd']);
 79 
 80         $base=$data['params'];
 81         $dum=explode(" ",$base);
 82         $light=$dum[1];
 83         $address=$dum[0]+1;
 84 //      var_dump($result);
 85 //      var_dump($dum[0]);
 86         $address_10=intval($address,10);
 87         $address_int=dechex($address_10);
 88         $light_10=intval($light,10);
 89         $light_int=dechex($light_10);
 90 //      var_dump($light_int);
91 //      var_dump($address_int);
 92         $adder=0xEF+0xFF+0x01+0x06+0x10+0x02+0xFF+0xFF+$address_10;
 93 //      var_dump($adder);
 94         $total=$adder+$light_10;
 95 //      var_dump($total);
 96         $init=floor($total/256);
 97         $rem=dechex($total%256);
 98 //      var_dump($rem);
 99 //      var_dump($init);
100         $li=pack('H*','efff0100061002ffff').pack('hH2hH2',$address_int,$light_int,$init,$rem).pack('H*','fe');
101         $serv->send($ffd,$li);
102         sleep(2);
103         $serv->send($ffd,$li);
104         sleep(2);
105         $serv->send($ffd,$li);
106         $serv->getLastError();
107         return "finished";
108 
109    }
110 
111    public function onFinish( $serv,$task_id,$data){
112 //      echo "Task is {$task_id} finished\n";
113 //      var_dump($data);
114 
115    }
116 
117    public function onTCPReceive(swoole_server $serv,$fd,$from_id,$data){
118         echo "fd is {$fd} aeecept from {$from_id} reactor\n";
119         var_dump($data);
120         $pd=substr($data,0,4);
121         var_dump($pd);
122         $zh=$pd.'-#*';
123         var_dump($zh);
124         $serv->send($fd,$zh);
125         $serv->getLastError();
126    }
127 
128 
129 }
130 new server();
131 ?>

