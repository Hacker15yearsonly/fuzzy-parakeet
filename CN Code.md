#TCP

//server

#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main() {
    int s, c;
    char b[1024];
    struct sockaddr_in a={0}, cl;
    socklen_t l=sizeof(cl);

    s=socket(AF_INET,SOCK_STREAM,0);
    a.sin_family=AF_INET;
    a.sin_addr.s_addr=INADDR_ANY;
    a.sin_port=htons(8080);
    bind(s,(void*)&a,sizeof(a));
    listen(s,5);

    printf("Server waiting for connection...\n");
    c=accept(s,(void*)&cl,&l);
    recv(c,b,1023,0);
    printf("Message from client: %s\n",b);

    strcpy(b,"Hello from server");
    send(c,b,strlen(b)+1,0);
    close(c);
    close(s);
}

//client

#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main() {
    int s=socket(AF_INET,SOCK_STREAM,0);
    char b[1024];
    struct sockaddr_in a={0};

    a.sin_family=AF_INET;
    a.sin_port=htons(8080);
    a.sin_addr.s_addr=inet_addr("127.0.0.1");
    connect(s,(void*)&a,sizeof(a));

    strcpy(b,"Hello from client");
    send(s,b,strlen(b)+1,0);
    recv(s,b,1023,0);
    printf("Message from server: %s\n",b);
    close(s);
}

#UDP

//server.c

#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main() {
    int s;
    char b[1024];
    struct sockaddr_in server, client;
    socklen_t len = sizeof(client);

    s = socket(AF_INET, SOCK_DGRAM, 0);
    memset(&server, 0, sizeof(server));

    server.sin_family = AF_INET;
    server.sin_port = htons(8080);
    server.sin_addr.s_addr = INADDR_ANY;

    if (bind(s, (struct sockaddr *)&server, sizeof(server)) < 0) {
        perror("Bind failed");
        return 1;
    }

    printf("UDP Server is running...\n");

    while (1) {
        int n = recvfrom(s, b, 1023, 0,
                         (struct sockaddr *)&client, &len);
        b[n] = '\0';
        printf("Client: %s\n", b);

        sendto(s, "Hello from UDP Server", 21, 0,
               (struct sockaddr *)&client, len);
    }
}

//client.c

#include <stdio.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>

int main() {
    int s, n;
    char b[1024];
    struct sockaddr_in server;
    socklen_t len = sizeof(server);

    s = socket(AF_INET, SOCK_DGRAM, 0);
    memset(&server, 0, sizeof(server));

    server.sin_family = AF_INET;
    server.sin_port = htons(8080);
    server.sin_addr.s_addr = inet_addr("127.0.0.1");

    sendto(s, "Hello from UDP Client", 21, 0,
           (struct sockaddr *)&server, len);
    printf("Message sent to server.\n");

    n = recvfrom(s, b, 1023, 0,
                 (struct sockaddr *)&server, &len);
    b[n] = '\0';
    printf("Server: %s\n", b);

    close(s);
}

#concurrent

//server

#include <stdio.h>
#include <time.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <pthread.h>
#include <stdlib.h>

int s;

void *sendtime(void *p) {
    struct sockaddr_in *c = p;
    char b[100];
    time_t t = time(NULL);
    snprintf(b, sizeof(b), "%s", ctime(&t));
    sendto(s, b, sizeof(b), 0, (void *)c, sizeof(*c));
    free(c);
    return NULL;
}

int main() {
    struct sockaddr_in a, c;
    socklen_t l;
    char b[100];
    pthread_t t;

    s = socket(AF_INET, SOCK_DGRAM, 0);
    a = (struct sockaddr_in){.sin_family=AF_INET,
        .sin_addr.s_addr=INADDR_ANY, .sin_port=htons(5000)};
    bind(s, (void *)&a, sizeof(a));
    printf("Time Server Started...\n");

    while (1) {
        l = sizeof(c);
        recvfrom(s, b, sizeof(b), 0, (void *)&c, &l);
        printf("Request received\n");
        struct sockaddr_in *p = malloc(sizeof(c));
        *p = c;
        pthread_create(&t, NULL, sendtime, p);
        pthread_detach(t);
    }
}

//Client

#include <stdio.h>
#include <arpa/inet.h>
#include <unistd.h>

int main() {
    int s = socket(AF_INET, SOCK_DGRAM, 0);
    struct sockaddr_in a = {.sin_family=AF_INET,
        .sin_port=htons(5000)};
    char b[100];
    socklen_t l = sizeof(a);

    a.sin_addr.s_addr = inet_addr("127.0.0.1");
    sendto(s, "time", 4, 0, (void *)&a, l);
    int n = recvfrom(s, b, 99, 0, (void *)&a, &l);
    b[n] = '\0';
    printf("Server time: %s\n", b);
    close(s);
}

#file

//server.c
#include <stdio.h>
#include <unistd.h>
#include <arpa/inet.h>

int main()
{
    int s, c, n;
    char b[1024];
    struct sockaddr_in a = {AF_INET, htons(8080), INADDR_ANY};

    s = socket(AF_INET, SOCK_STREAM, 0);
    bind(s, (void *)&a, sizeof(a));
    listen(s, 1);

    printf("Server waiting for a client...\n");
    c = accept(s, NULL, NULL);

    FILE *f = fopen("received_file", "wb");
    while ((n = recv(c, b, sizeof(b), 0)) > 0)
        fwrite(b, 1, n, f);

    fclose(f);
    close(c);
    close(s);
    printf("File received successfully as received_file\n");
}

//client.c

#include <stdio.h>
#include <unistd.h>
#include <arpa/inet.h>

int main()
{
    int s, n;
    char ip[20], file[100], b[1024];
    struct sockaddr_in a = {AF_INET, htons(8080)};

    printf("Enter server IP: ");
    scanf("%19s", ip);
    printf("Enter file name: ");
    scanf("%99s", file);

    FILE *f = fopen(file, "rb");
    if (!f) return 1;

    s = socket(AF_INET, SOCK_STREAM, 0);
    a.sin_addr.s_addr = inet_addr(ip);
    connect(s, (void *)&a, sizeof(a));

    while ((n = fread(b, 1, sizeof(b), f)) > 0)
        send(s, b, n, 0);

    shutdown(s, SHUT_WR);
    fclose(f);
    close(s);
    printf("File sent successfully.\n");
}

#Packet

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <net/ethernet.h>
#include <netinet/ip.h>
#include <sys/socket.h>

#define SIZE 65536

int main()
{
    int sock, count = 0, n;
    unsigned char buffer[SIZE];

    sock = socket(AF_PACKET, SOCK_RAW, htons(ETH_P_ALL));
    if (sock < 0) {
        perror("Socket Creation Failed (Run with sudo)");
        exit(1);
    }

    printf("Raw Socket Packet Sniffer started... Press Ctrl+C to stop.\n");

    while (count < 5) {
        n = recvfrom(sock, buffer, SIZE, 0, NULL, NULL);
        if (n < 0) continue;

        struct ethhdr *eth = (struct ethhdr *)buffer;
        count++;

        printf("\n<<< Captured Packet #%d [%d Bytes] >>>\n", count, n);
        printf("\n================ Ethernet Frame ================\n");

        printf("Destination MAC : %.2X-%.2X-%.2X-%.2X-%.2X-%.2X\n",
            eth->h_dest[0], eth->h_dest[1], eth->h_dest[2],
            eth->h_dest[3], eth->h_dest[4], eth->h_dest[5]);

        printf("Source MAC      : %.2X-%.2X-%.2X-%.2X-%.2X-%.2X\n",
            eth->h_source[0], eth->h_source[1], eth->h_source[2],
            eth->h_source[3], eth->h_source[4], eth->h_source[5]);

        printf("Protocol        : 0x%.4X\n", ntohs(eth->h_proto));

        if (ntohs(eth->h_proto) == ETH_P_IP) {
            struct iphdr *ip = (struct iphdr *)(buffer + sizeof(struct ethhdr));
            struct in_addr src = {ip->saddr}, dst = {ip->daddr};

            printf("----------------- IP Header -----------------\n");
            printf("IP Version      : %d\n", ip->version);
            printf("Header Length   : %d Bytes\n", ip->ihl * 4);
            printf("TTL             : %d\n", ip->ttl);
            printf("Protocol        : %d\n", ip->protocol);
            printf("Source IP       : %s\n", inet_ntoa(src));
            printf("Destination IP  : %s\n", inet_ntoa(dst));
            printf("================================================\n");
        } else {
            printf("Non-IPv4 Frame Captured.\n");
        }
    }

    close(sock);
    printf("\nPacket capture complete.\n");
    return 0;
}
