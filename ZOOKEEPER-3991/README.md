## Summary

- The DNS resolution was not retrying while the Zookeeper is executed and the old failed result used to be used.

## Metadata

- Bug report : <https://issues.apache.org/jira/browse/ZOOKEEPER-3991>
- PR containing fix : <https://github.com/apache/zookeeper/pull/1524>
- The Guideline to test : <https://cwiki.apache.org/confluence/display/ZOOKEEPER/HowToContribute>
- Retry bug category : HOW

## Findings

- The failure happened because the InetSocketAddress is created once and the DNS lookup happens when it is created.
- This is fixed by adding isUnresolved() to the address param :

```java
if (address.isUnresolved()) {
                    // Retry DNS resolution
          address = new InetSocketAddress(address.getHostName(), address.getPort());
    }
```

- The retry loop is located in HiveConnection's constructor

```java
 while ((!shutdown) && (portBindMaxRetry == 0 || numRetries < portBindMaxRetry)) {
                    try {
                        ...
                        while (!shutdown) {
                            try {
                                client = serverSocket.accept();
                                setSockOpts(client);
                        // the code lines under this, make the retries while trying to keep throwing unresolved address errors even though the DNS entry became correct
                                if (quorumSaslAuthEnabled) {
                                    receiveConnectionAsync(client);
                                } else {
                                    receiveConnection(client);
                                }
                                numRetries = 0;
                            } catch (SocketTimeoutException e) {
                                ...
                            }
                        }
                    } catch (IOException e) {
                        if (shutdown) {
                            break;
                        }
                        if (e instanceof SocketException) {
                            socketException.set(true);
                        }

                        numRetries++;
                        try {
                            close();
                            Thread.sleep(1000);
                        } catch (IOException ie) {
                            ...
                        } catch (InterruptedException ie) {
                            ...
                        }
                        closeSocket(client);
                    }
                }
                if (!shutdown) {
                    ...
                }
```

## Steps to Run Tests

1.  Clone the Zookeeper Repository :

```
git clone <https://github.com/<userid>/zookeeper.git> my-zookeeper
```

# Unit Tests

2.  Build ZooKeeper using Maven :

```
mvn clean install -DskipTests
```

3.  Run the unit tests :

```
mvn test
```

4. Search for '[GarudaACE] - QuorumCnxManager' by using this command. For unit test:

```
find -path '*/surefire-reports/*-output.txt' -exec grep -lr "\[GarudaACE] - QuorumCnxManager" {} +
```

for MacOS and create into new folder

```
find . -path '*surefire-reports/*-output.txt' -exec grep -q "\[GarudaACE\] - QuorumCnxManager" {} \; -exec sh -c 'mkdir -p new_folder && cp -p "{}" new_folder/' \;
```

5. For running a specific test class or test method, we can use the `-Dtest` option to specify the class or method to run. For example, to run only the test method `testPingWithWatcher` in the `org.apache.zookeeper.ClientTest` class

```
mvn test -Dtest=org.apache.zookeeper.ClientTest#testPingWithWatcher
```

6. Once the tests have completed running, we can view the Surefire reports by navigating to the `target/surefire-reports` directory. Then, look at a set of HTML files that provide detailed information on the test results.
