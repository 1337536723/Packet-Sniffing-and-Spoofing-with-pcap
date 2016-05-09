all: icmp4.c tcp4.c udp4_ll.c 
	gcc -Wall -o icmp4 icmp4.c -lpcap 
	gcc -Wall -o tcp4 tcp4.c -lpcap 
	gcc -Wall -o udp4_ll udp4_ll.c -lpcap 
    
clean:  
	rm -rf *.o test 
