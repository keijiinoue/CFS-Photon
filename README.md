# Photon で CFS に接続する方法

Particle社製 Photon を使って Connected Field Service for Dynamics 365 (CFS)  に接続する方法のサンプルです。  

## 動作確認環境
* Particle社製 Photon Kit （2017年1月に購入）
* Dynamics 365 (CRM) Version 1612 (8.2.0.773)

## スコープ
* Photon で明るさに基づくデータを Azure IoT 側に送信します。
* Photon で閾値以上の明るさを検知すると、Azure IoT を経由して異常値と判断し、Dynamics 365 側に IoT 通知レコードが作成さます。
* CFS が元々持っているデバイスのシミュレーターのデータもAzure IoT および Dynamcis 365 側でハンドリングできます。
* CFS が元々持っている IoT 通知レコード作成後のサポート案件や作業指示書などの後続シナリオ、ダッシュボード、および Power BI 連携も可能です。
* **注意： デバイスへのコマンド送信は対象としていません。**

## 手順
1. CFS の初期セットアップ
	* 以下のページを参考にセットアップをします。  
		Use Connected Field Service to remotely monitor and service customer equipment (field service)  
		https://www.microsoft.com/en-US/dynamics/crm-customer-center/use-connected-field-service-to-remotely-monitor-and-service-customer-equipment-field-service.aspx
	* デバイスのシミュレーターから異常値（70度以上）を発生させて、IoT 通知レコードが生成されることを確認ください。

1. Photon の初期セットアップ
	* 以下のページを参考に、基本的な動作ができるようにセットアップをします。  
		Getting Started  
		https://docs.particle.io/guide/getting-started/start/photon/  
	* 次に、Particle社の Web IDE (Build) の使い方にも慣れて頂いた上で、以下のページにある明るさを検知してそれが数値化される仕組みをセットアップします。  
		Read your Photoresistor: Function and Variable  
		https://docs.particle.io/guide/getting-started/examples/photon/#read-your-photoresistor-function-and-variable

1. CFS と Photon の初期連携方法のセットアップ
	* 以下の LMS サイトでの、コース「005. Connected Field Service」を受講して、Particle社の製品と CFS を接続する方法を実装します。  
		Microsoft Field Service & Project Service Automation LMS  
		https://fieldone.litmos.com/  
	* この段階では、CFS が元々持っているデバイスのシミュレーターのデータが反映されない状態です。  

1. Photon で温度データとして出力するためのサンプル プログラムのセットアップ
	CFS は既定で70度以上の温度データを異常値と見なします。Photon Kit では明るさを検知できます。デモ上はそれを適切な温度データであると見なして数値変換して、Azure IoT 側に出力するようにします。  
	以下のプログラムを Particle Web IDE (Build) で、例えば「photoresistor-alert.ino」というような名前で作成し、Photon に flash します。  

	```
	int led1 = D0;
	int led2 = D7;
	int photoresistor = A0;
	int power = A5;
	
	float t_normal = 18.0;
	float br_normal = 360.0;
	float t_high = 70.0;
	float br_high = 1200.0;
	
	// Like y = ax + b, a and b are coefficient.
	// t = abr + b, t = temperature, br = brightness
	float a = (t_high - t_normal) / (br_high - br_normal);
	float b = t_normal - a * br_normal;
	
	void setup() {
	  pinMode(led1, OUTPUT);
	  pinMode(led2, OUTPUT);
	  pinMode(photoresistor,INPUT);
	  pinMode(power,OUTPUT);
	
	  analogWrite(power,255);
	}
	
	void loop() {
	  int brightness = analogRead(photoresistor);
	  float temperature = a * brightness + b;
	
	  Particle.publish("temperature", String((int)temperature), 60, PRIVATE);
	
	  if((int)temperature < 70){
	    normalBlink();
	  }else{
	    alertingBlink();
	  }
	}
	
	// takes 6 seconds
	void normalBlink(){
	  digitalWrite(led2, LOW);
	
	  for(int i=0; i<3; i++){
	    digitalWrite(led1, HIGH);
	    delay(1500);
	    digitalWrite(led1, LOW);
	    delay(500);
	  }
	}
	
	// takes 6 seconds
	void alertingBlink(){
	  digitalWrite(led2, HIGH);
	
	  for(int i=0; i<6; i++){
	    digitalWrite(led1, HIGH);
	    delay(800);
	    digitalWrite(led1, LOW);
	    delay(200);
	  }
	}
	```

1. Photon のデータもシミュレーターのデータも反映させるための Azure Stream Analytics ジョブの設定
	CFS セットアップの際に指定したリソース グループを開きます。  
	2つの Stream Analytics ジョブがあり、2つとも編集します。  
	1. 出力が AlertsQueue になっているものを選択します。それを「停止」します。以下のようにクエリを編集します。  

		```
		WITH AlertData AS 
		(
		    SELECT
		        CASE
		            WHEN Stream.device_id IS NOT null THEN Stream.device_id
		            ELSE Stream.DeviceID
		        END as DeviceID,
		         'Temperature' AS ReadingType,
		         CASE
		            WHEN Stream.data IS NOT null THEN Stream.data
		            ELSE Stream.Temperature
		        END as Reading,
		         Stream.EventToken AS EventToken,
		         Ref.Temperature AS Threshold,
		         Ref.TemperatureRuleOutput AS RuleOutput,
		         Stream.EventEnqueuedUtcTime AS [Time]
		    FROM IoTStream Stream
		    JOIN DeviceRulesBlob Ref ON Ref.DeviceType = 'Thermostat'
		    WHERE
		         Ref.Temperature IS NOT null AND
		            CASE
		                WHEN Stream.data IS NOT null THEN CAST( Stream.data as bigint )
		                ELSE Stream.Temperature
		            END 
	                > Ref.Temperature
		)
		
		SELECT data.DeviceId,
		    data.ReadingType,
		    data.Reading,
		    data.EventToken,
		    data.Threshold,
		    data.RuleOutput,
		    data.Time
		INTO AlertsQueue
		FROM AlertData data
		WHERE LAG(data.DeviceID) OVER (PARTITION BY data.DeviceId, data.Reading, data.ReadingType LIMIT DURATION(minute, 1)) IS NULL
		```

	1. 出力が PowerBISQL になっているものを選択します。それを「停止」します。以下のようにクエリを編集します。  

		```
		WITH TelemetryData AS 
		(
			SELECT
			    CASE
				    WHEN Stream.device_id IS NOT null THEN Stream.device_id
				    ELSE Stream.DeviceID
			    END as DeviceID,
			     'Temperature' AS ReadingType,
			    CASE
				    WHEN Stream.data IS NOT null THEN CAST( Stream.data as bigint )
				    ELSE Stream.Temperature
			    END as Reading,
			     Stream.EventToken AS EventToken,
			     Ref.Temperature AS Threshold,
			     Ref.TemperatureRuleOutput AS RuleOutput,
			     Stream.EventEnqueuedUtcTime AS [Time]
			FROM IoTStream Stream
			JOIN DeviceRulesBlob Ref ON Ref.DeviceType = 'Thermostat'
		),
		MaxInMinute AS
		(
			SELECT
			    TopOne() OVER (ORDER BY Reading DESC) AS telemetryEvent
			FROM
			    TelemetryData 
			GROUP BY 
			    TumblingWindow(minute, 1), DeviceId
		)
		
		SELECT telemetryEvent.DeviceId,
		    telemetryEvent.ReadingType,
		    telemetryEvent.Reading,
		    telemetryEvent.EventToken,
		    telemetryEvent.Threshold,
		    telemetryEvent.RuleOutput,
		    telemetryEvent.Time
		INTO PowerBISQL
		FROM MaxInMinute
		```

	以上です。

