# Using an external configuration file while running containerized JBoss EAP or WildFly
The following walkthrough aims to document the possibility of 'injecting' external JBoss/Wildfly configuration files when running a containerized application server.

The documented passages are done using
- quay.io as image registry
- podman as the container engine

Test have been run on 2019 Intel MBP with Ventura (additional MacOS related settings for podman are not mentioned here).

## Standard run of the container image (default configuration file)
To start, firstly run the publicly available wildfly image from quay.io

```bash
podman run quay.io/wildfly/wildfly 
```
The application server will start and print INFO level console logs as configured in the default configuration file used, that can be found in the container at the path

```bash
/opt/jboss/wildfly/standalone/configuration/standalone.xml
```

Once this has been verified, stop the running container by finding the conatinerID found using 

```
podman ps
```
and then terminating it by

```podman
podman stop <containerID>
```

## Run the container image by using a different configuration file
Create a modified version of standalone.xml changing the console log level by modifying the CONSOLE console-handler and setting its level to TRACE

```xml
            <console-handler name="CONSOLE">
                <level name="TRACE"/>
                <formatter>
                    <named-formatter name="COLOR-PATTERN"/>
                </formatter>
            </console-handler>
```
Now run the same container image by binding a volume with the modified file (in the current directory) to the same path of the original configuration file:

```bash
podman run -v ./standalone.xml:/opt/jboss/wildfly/standalone/configuration/standalone.xml quay.io/wildfly/wildfly
```

By looking at the console output the log level has been updated by the externally mounted file and this can be further verified by copying locally the file injected in the container by using podman

```bash
podman cp <containerID>:/opt/jboss/wildfly/standalone/configuration/standalone.xml ./standalone-from-container.xml
```

## Testing the management CLI and verify configuration persistency
Running wildlfly in a container still allows to run the jboss CLI to manage the application server live.
Connect to the container e start the cli by doing:

```bash
podman exec -i  <containerID> ./wildfly/bin/jboss-cli.sh
```

The local terminal will connect to the container and start the connection and allow to run commands to inspect and change the current configurations and verify the changes

```bash
/subsystem=logging/console-handler=CONSOLE:read-resource
/subsystem=logging/console-handler=CONSOLE:write-attribute(name="level", value="INFO")
/subsystem=logging/console-handler=CONSOLE:read-resource
```

If the container is configured to be able to write to the volume it would be also possible to persist the changes, if that's not the case the changes will still be effective on the server, but the external file won't be updated.
