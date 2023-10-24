# VMware Cross vCenter Workload Migration Utility
# https://flings.vmware.com/cross-vcenter-workload-migration-utility

FROM adoptopenjdk/openjdk8
RUN mkdir /app
ADD https://download3.vmware.com/software/vmw-tools/xvc-mobility-cli/xvm-3.1.jar /app/
WORKDIR /app
CMD java -jar -Dserver.port=8443 /app/xvm-3.1.jar
EXPOSE 8443/tcp
