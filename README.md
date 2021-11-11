# Kafka 2-Way SSL Authentication Demo

Simple setup to demonstrate TLS (aka SSL) communication between brokers and clients using spring.


To run you need OpenSSL (to generate and sign certificates), Java 8 and Docker installed.

```
- For the next steps, issue all commands in the folder ./certs

- If it not exists, create the folders ./certs/keystore and ./certs/truststore

- Whenever asked for "cn" or "first last name" input "localhost"

- To simplify the configuration keep all passwords the same
```
## Server Configuration

#### 1. Generate the CA to self sign the certificates

```sh
openssl req -new -x509 -keyout ca-key -out ca-cert -days 365
```
This generates the files ```ca-cert``` and ```ca-key```

#### 2. Generate keys and certificates for the Kafka Broker

```sh
keytool -keystore keystore/kafka.server.keystore.jks -alias localhost -keyalg RSA -genkey
```
This generates the file ```keystore/kafka.server.keystore.jks```

#### 3. Add the generated CA to the brokers’ truststore so that the brokers can trust this CA.

```sh
keytool -keystore truststore/kafka.server.truststore.jks -alias CARoot -importcert -file ca-cert
```

#### 4. Export the certificate from the Kafka Broker keystore, generated above

```sh
keytool -keystore keystore/kafka.server.keystore.jks -alias localhost -certreq -file cert-file-server
```
This generates the file ```cert-file-server```

#### 5. Sign the cert-file-server using the CA created above

```sh
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file-server -out cert-server-signed -days 365 -CAcreateserial
```
This generates the file ```cert-server-signed```

#### 6. Import both the certificate of the CA and the signed certificate into the Kafka Broker keystore

```sh
keytool -keystore keystore/kafka.server.keystore.jks -alias CARoot -importcert -file ca-cert
keytool -keystore keystore/kafka.server.keystore.jks -alias localhost -importcert -file cert-server-signed
```

#### 7. Create a credentials file for the Kafka Broker

```sh
echo [password] > credentials
```
Where ```[password]``` was the password you used so far.
If you didn't use the same password when asked for one, you need to create one credential file for each one and adjust the ```docker-compose.yml``` to reflect that.

#### 8. Check Kafka Configuration

At this point you have all you need to run the kafka docker-compose in the project without errors.
Now we are going to issue a command to make sure everything is OK so far.

Run the docker-compose:

```sh
docker-compose up
```

Wait until it is up and check if didn't show any error messages.

Also check in the logs (docker-compose sysout) for the points:

```
advertised.listeners = SSL://localhost:9093
listeners = SSL://0.0.0.0:9093
security.inter.broker.protocol = SSL
ssl.protocol = TLSv1.3
ssl.client.auth = required
ssl.keystore.location = /etc/kafka/secrets/certs/keystore/kafka.server.keystore.jks
ssl.truststore.location = /etc/kafka/secrets/certs/truststore/kafka.server.truststore.jks

INFO Registered broker 1 at path /brokers/ids/1 with addresses: SSL://localhost:9093
```
If there is something different for these parameters it is better to revise the config again before continue.


Now, issue the command to verify SSL handshake error (try to connect without cert/auth):

```sh
openssl s_client -connect localhost:9093
```

You will see an output like this:

```
.
. < info ommited >
.
Server certificate
-----BEGIN CERTIFICATE-----
.
. < ommited certificate stuff >
.
-----END CERTIFICATE-----

.
. <more stuff ommited>
.

140007831230272:error:1408F119:SSL routines:ssl3_get_record:decryption failed or bad record mac:../ssl/record/ssl3_record.c:676:

```
Note the "decryption failed" message.

And in docker-compose sysout logs you will see:

```
INFO [SocketServer listenerType=ZK_BROKER, nodeId=1] Failed authentication with /<some ip> (SSL handshake failed) (org.apache.kafka.common.network.Selector)
```

Now let's pass the certificate and key:

```sh
openssl s_client -connect localhost:9093 -cert ca-cert -key ca-key
```

And we should get the output and no handshake errors in docker-compose sysout:

```
Post-Handshake New Session Ticket arrived:
SSL-Session:
Protocol  : TLSv1.3
Cipher    : TLS_AES_256_GCM_SHA384
Session-ID: 0E971B8151C41ABB246922CC9090B82A6FCFD0CA4A8D23DDB441174AD73848EB
```

And we're done! We have a Kafka Server configured to accept 2-Way SSL Authentication connections.

## Client Configuration

Now for the easy part, we need to generate both truststore and keystore for the clients to be able to communicate with our server.

#### 1. Create the Client Keystore

```sh
keytool -keystore keystore/kafka.client.keystore.jks -alias localhost -keyalg RSA -genkey
```
This generates the file ```keystore/kafka.client.keystore.jks```

#### 2. Add the generated CA to the clients’ truststore so that the clients can trust this CA

```sh
keytool -keystore truststore/kafka.client.truststore.jks -alias CARoot -importcert -file ca-cert
```
This generates the file ```truststore/kafka.client.truststore.jks```

#### 3. Export the certificate from the keystore

```sh
keytool -keystore keystore/kafka.client.keystore.jks -alias localhost -certreq -file cert-file-client
```
This generates the file ```cert-file-client```

#### 4. Sign the cert-file-client using the CA 

```sh
openssl x509 -req -CA ca-cert -CAkey ca-key -in cert-file-client -out cert-client-signed -days 365 -CAcreateserial
```
This generates the file ```cert-client-signed```

#### 5. Import both the certificate of the CA and the signed certificate into the client keystore

```sh
keytool -keystore keystore/kafka.client.keystore.jks -alias CARoot -importcert -file ca-cert
keytool -keystore keystore/kafka.client.keystore.jks -alias localhost -importcert -file cert-client-signed
```

And we're done!

Now we can use ```keystore/kafka.client.keystore.jks``` and ```truststore/kafka.client.truststore.jks``` in our clients to connect to our server.

You can run the sample spring application to see it producing and consuming messages, or you can fire up Kafka Tool and test the connection using the interface.
I strong suggest that you switch between ```KakfaConfiguration``` and ```KakfaConfigurationNoSSL``` configurations to see the impact of enabling SSL.

Another way to test is using own kafka producer/consumer utils as you can see [here](https://docs.confluent.io/platform/current/security/security_tutorial.html#configure-clients), you just need to omit the SASL auth part.


For more detailed in depth configuration and added authentication security visit [Kafka Confluent](https://docs.confluent.io/platform/current/security/security_tutorial.html) tutorial page.
