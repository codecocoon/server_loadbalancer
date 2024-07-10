# server_loadbalancer

``` cpp
#include <iostream>
#include <vector>
#include <set>
#include <chrono>
#include <thread>
#include <memory>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <random>

const int minConnections = 0;
const int maxConnections = 1000;
const int maxRandomCandidates = 10;

struct Server {
    int channelId;
    int connections;
    bool isOpen;

    Server(int channelId, int connections, bool isOpen)
        : channelId(channelId), connections(connections), isOpen(isOpen) {}

    bool operator<(const Server& other) const {
        if (connections == other.connections) {
            return channelId < other.channelId;
        }
        return connections < other.connections;
    }
};

class LoadBalancer {
private:
    std::set<Server> serverSet;
    std::mt19937 rng;

public:
    LoadBalancer(const std::vector<Server>& backendServers) {
        std::random_device rd;
        rng.seed(rd());

        for (const auto& server : backendServers) {
            if (server.isOpen) {
                serverSet.insert(server);
            }
        }
    }

    Server getServerWithLeastConnections() {
        std::vector<Server> minConnectionServers;
        int minConnections = maxConnections + 1;

        for (const auto& server : serverSet) {
            if (server.isOpen) {
                if (server.connections < minConnections) {
                    minConnectionServers.clear();
                    minConnectionServers.push_back(server);
                    minConnections = server.connections;
                } else if (server.connections == minConnections && minConnectionServers.size() < maxRandomCandidates) {
                    minConnectionServers.push_back(server);
                }

                if (minConnectionServers.size() == maxRandomCandidates) {
                    break;
                }
            }
        }

        if (minConnectionServers.empty()) {
            std::cout << "No open servers available" << std::endl;
            return Server(-1, -1, false); // Invalid server
        }

        std::uniform_int_distribution<std::size_t> dist(0, minConnectionServers.size() - 1);
        return minConnectionServers[dist(rng)];
    }

    void incrementConnections(Server& server) {
        serverSet.erase(server);
        if (server.connections < maxConnections) {
            ++server.connections;
        }
        serverSet.insert(server);
    }

    void decrementConnections(Server& server) {
        serverSet.erase(server);
        if (server.connections > minConnections) {
            --server.connections;
        }
        serverSet.insert(server);
    }

    void setServerOpenState(int channelId, bool isOpen) {
        for (auto it = serverSet.begin(); it != serverSet.end(); ++it) {
            if (it->channelId == channelId) {
                Server updatedServer = *it;
                serverSet.erase(it);
                updatedServer.isOpen = isOpen;
                if (isOpen) {
                    serverSet.insert(updatedServer);
                }
                break;
            }
        }
    }

    void printLoadBalancerState() {
        std::cout << "Current load balancer state:" << std::endl;
        for (const auto& server : serverSet) {
            std::cout << "Server " << server.channelId << " - Connections: " << server.connections << " - IsOpen: " << server.isOpen << std::endl;
        }
        std::cout << std::endl;
    }
};

void producer(std::shared_ptr<LoadBalancer>& lb, std::mutex& lbMutex, std::condition_variable& cv, bool& stopFlag) {
    while (!stopFlag) {
        std::vector<Server> backendServers;
        for (int i = 1; i <= 1000; ++i) {
            backendServers.push_back({i, 0, true});
        }

        auto start = std::chrono::high_resolution_clock::now();
        auto newLb = std::make_shared<LoadBalancer>(backendServers);
        auto end = std::chrono::high_resolution_clock::now();
        auto creation_duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count();
        std::cout << "Time to create LoadBalancer: " << creation_duration << " nanoseconds" << std::endl;

        std::cout << "======================" << std::endl;

        {
            std::lock_guard<std::mutex> lock(lbMutex);
            lb = newLb;
        }
        cv.notify_one();

        std::this_thread::sleep_for(std::chrono::seconds(30));
    }
}

void consumer(std::shared_ptr<LoadBalancer>& lb, std::mutex& lbMutex, std::condition_variable& cv, bool& stopFlag) {
    while (!stopFlag) {
        std::shared_ptr<LoadBalancer> currentLb;

        {
            std::unique_lock<std::mutex> lock(lbMutex);
            if (!lb) {
                cv.wait(lock, [&lb, &stopFlag] { return lb || stopFlag; });
            }
            currentLb = lb;
        }

        if (currentLb) {
            int iterations = 1000;

            auto start = std::chrono::high_resolution_clock::now();
            for (int i = 0; i < iterations; ++i) {
                Server server = currentLb->getServerWithLeastConnections();
                if (server.channelId == -1) {
                    break;
                }
            }
            auto end = std::chrono::high_resolution_clock::now();
            auto select_duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count() / iterations;

            start = std::chrono::high_resolution_clock::now();
            for (int i = 0; i < iterations; ++i) {
                Server selectedServer = currentLb->getServerWithLeastConnections();
                if (selectedServer.channelId == -1) {
                    break;
                }
                currentLb->incrementConnections(selectedServer);
            }
            end = std::chrono::high_resolution_clock::now();
            auto increment_duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count() / iterations;

            start = std::chrono::high_resolution_clock::now();
            for (int i = 0; i < iterations; ++i) {
                Server selectedServer = currentLb->getServerWithLeastConnections();
                if (selectedServer.channelId == -1) {
                    break;
                }
                currentLb->decrementConnections(selectedServer);
            }
            end = std::chrono::high_resolution_clock::now();
            auto decrement_duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count() / iterations;

            start = std::chrono::high_resolution_clock::now();
            for (int i = 0; i < iterations; ++i) {
                Server selectedServer = currentLb->getServerWithLeastConnections();
                if (selectedServer.channelId == -1) {
                    break;
                }
                currentLb->incrementConnections(selectedServer);
            }
            end = std::chrono::high_resolution_clock::now();
            auto select_and_increment_duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count() / iterations;

            start = std::chrono::high_resolution_clock::now();
            for (int i = 0; i < iterations; ++i) {
                Server selectedServer = currentLb->getServerWithLeastConnections();
                if (selectedServer.channelId == -1) {
                    break;
                }
                currentLb->decrementConnections(selectedServer);
            }
            end = std::chrono::high_resolution_clock::now();
            auto select_and_decrement_duration = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count() / iterations;

            std::cout << "Average time to select server: " << select_duration << " nanoseconds" << std::endl;
            std::cout << "Average time to increment server connections: " << increment_duration << " nanoseconds" << std::endl;
            std::cout << "Average time to decrement server connections: " << decrement_duration << " nanoseconds" << std::endl;
            std::cout << "Average time to select and increment server connections: " << select_and_increment_duration << " nanoseconds" << std::endl;
            std::cout << "Average time to select and decrement server connections: " << select_and_decrement_duration << " nanoseconds" << std::endl;
        }

        std::this_thread::sleep_for(std::chrono::seconds(1)); // 다음 작업을 위해 1초 대기
    }
}

int main() {
    std::shared_ptr<LoadBalancer> lb = nullptr;
    std::mutex lbMutex;
    std::condition_variable cv;
    bool stopFlag = false;

    std::thread producerThread(producer, std::ref(lb), std::ref(lbMutex), std::ref(cv), std::ref(stopFlag));
    std::thread consumerThread(consumer, std::ref(lb), std::ref(lbMutex), std::ref(cv), std::ref(stopFlag));

    std::this_thread::sleep_for(std::chrono::minutes(5));
    stopFlag = true;
    cv.notify_all();

    producerThread.join();
    consumerThread.join();

    return 0;
}
```
