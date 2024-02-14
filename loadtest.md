### MeshCircuitBreaker

kkd exec -it <fe-pod> sh


BACKEND_URL="http://backend:3001"

CONNECTIONS=100

hold_connection() {
    curl -s --keepalive-time 5 "$BACKEND_URL" > /dev/null
    echo "Connection to $BACKEND_URL attempted"
}

for i in $(seq 1 $CONNECTIONS); do
    hold_connection &
done

wait
echo "Attempted $CONNECTIONS connections to $BACKEND_URL"


### MeshFaultInjection:

    for i in $(seq 1 100); do curl -s http://backend:3001 > /dev/null & done


### MeshRateLimit

curl -s http://backend:3001 (a few times to see the rate limit)

