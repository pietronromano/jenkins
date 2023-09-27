# Script pasted directly in Jenkins Freestyle UI

#!/bin/sh
file_name=time_log.txt
current_time=$(date "+%Y.%m.%d-%H.%M.%S")
echo "Writing to $current_time to $file_name"
echo "Current Time : $current_time" >> $file_name
echo "Getting User:"
whoami
echo "PATH:"
echo $PATH
echo "Getting Maven Version:"
mvn --version