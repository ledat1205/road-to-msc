![[Pasted image 20260111233245.png]]

**Cloud Pub/Sub**: is a global scale messaging buffer that decouples sender and receiver, Serverless 
Use case:![[Pasted image 20260111233733.png]]

**Kafka Connector**: allow seamless integration between kafka and pub/sub
![[Pasted image 20260111233907.png]]

![[Pasted image 20260112043323.png]]

Message lifecycle
- create topic
- publisher send message to that topic
- subscriber pull message or queue push message to subscriber
- subscriber acknowledgment received message
- pub/sub delete message (message retention)
![[Pasted image 20260112043705.png]]

![[Pasted image 20260112044020.png]]
