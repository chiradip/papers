#include <chrono>
#include <vector>
#include <unordered_map>
#include <memory>
#include <atomic>
#include <mutex>
#include <thread>
#include <queue>
#include <optional>
#include <functional>
#include <random>
#include <iostream>
#include <algorithm>

// Forward declarations
class CausalEvent;
class TrueTimeAPI;
class CCRPNode;

// ============================================================================
// TrueTime-inspired Temporal API
// ============================================================================

struct TimeInterval {
    std::chrono::steady_clock::time_point earliest;
    std::chrono::steady_clock::time_point latest;
    
    TimeInterval(std::chrono::steady_clock::time_point e, 
                 std::chrono::steady_clock::time_point l) 
        : earliest(e), latest(l) {}
    
    // Uncertainty in microseconds
    int64_t uncertainty_us() const {
        return std::chrono::duration_cast<std::chrono::microseconds>(
            latest - earliest).count();
    }
    
    bool contains(std::chrono::steady_clock::time_point t) const {
        return t >= earliest && t <= latest;
    }
};

class TrueTimeAPI {
private:
    std::atomic<int64_t> uncertainty_us_{1000}; // 1ms default uncertainty
    mutable std::mutex mutex_;
    
public:
    // Simulates TrueTime.now() with uncertainty bounds
    TimeInterval now() const {
        auto real_now = std::chrono::steady_clock::now();
        auto uncertainty = std::chrono::microseconds(uncertainty_us_.load());
        
        return TimeInterval(
            real_now - uncertainty,
            real_now + uncertainty
        );
    }
    
    // Simulates network/system conditions affecting uncertainty
    void set_uncertainty_us(int64_t us) {
        uncertainty_us_.store(us);
    }
    
    // Wait until we're sure 't' is in the past
    void wait_until_past(std::chrono::steady_clock::time_point t) const {
        auto current = now();
        if (current.earliest <= t) {
            auto wait_time = t - current.earliest + 
                           std::chrono::microseconds(uncertainty_us_.load());
            std::this_thread::sleep_for(wait_time);
        }
    }
};

// ============================================================================
// Causal Event and Vector Clock Implementation
// ============================================================================

using NodeId = uint32_t;
using EventId = uint64_t;
using VectorClock = std::unordered_map<NodeId, uint64_t>;

class CausalEvent {
public:
    EventId id;
    NodeId origin_node;
    std::chrono::steady_clock::time_point timestamp;
    VectorClock vector_clock;
    std::string data;
    std::vector<EventId> causal_dependencies;
    
    CausalEvent(EventId id, NodeId origin, const std::string& data)
        : id(id), origin_node(origin), data(data), 
          timestamp(std::chrono::steady_clock::now()) {}
    
    // Causal ordering: this event happened before 'other'
    bool happens_before(const CausalEvent& other) const {
        // Check if this event's vector clock is less than other's
        for (const auto& [node, clock] : vector_clock) {
            auto it = other.vector_clock.find(node);
            if (it == other.vector_clock.end() || clock > it->second) {
                return false;
            }
        }
        
        // At least one element must be strictly less
        for (const auto& [node, clock] : vector_clock) {
            auto it = other.vector_clock.find(node);
            if (it != other.vector_clock.end() && clock < it->second) {
                return true;
            }
        }
        
        return false;
    }
    
    bool is_concurrent_with(const CausalEvent& other) const {
        return !happens_before(other) && !other.happens_before(*this);
    }
};

// ============================================================================
// CCRP Node Implementation
// ============================================================================

class CCRPNode {
private:
    NodeId node_id_;
    std::shared_ptr<TrueTimeAPI> truetime_;
    
    // Event storage
    std::mutex events_mutex_;
    std::unordered_map<EventId, std::shared_ptr<CausalEvent>> events_;
    
    // Vector clock for this node
    std::mutex clock_mutex_;
    VectorClock local_clock_;
    
    // Recent Past Window configuration
    std::chrono::microseconds recent_past_window_{10000}; // 10ms default
    
    // Causal convergence tracking
    mutable std::mutex convergence_mutex_;
    std::unordered_map<std::chrono::steady_clock::time_point, 
                       std::vector<EventId>> time_buckets_;
    
public:
    CCRPNode(NodeId id, std::shared_ptr<TrueTimeAPI> tt) 
        : node_id_(id), truetime_(tt) {
        std::lock_guard<std::mutex> lock(clock_mutex_);
        local_clock_[node_id_] = 0;
    }
    
    // ========================================================================
    // Event Creation and Management
    // ========================================================================
    
    std::shared_ptr<CausalEvent> create_event(const std::string& data) {
        std::lock_guard<std::mutex> clock_lock(clock_mutex_);
        std::lock_guard<std::mutex> events_lock(events_mutex_);
        
        // Increment local clock
        local_clock_[node_id_]++;
        
        auto event = std::make_shared<CausalEvent>(
            local_clock_[node_id_], node_id_, data);
        event->vector_clock = local_clock_;
        
        events_[event->id] = event;
        
        // Add to time bucket for convergence tracking
        add_to_time_bucket(event->timestamp, event->id);
        
        return event;
    }
    
    void receive_event(std::shared_ptr<CausalEvent> event) {
        std::lock_guard<std::mutex> clock_lock(clock_mutex_);
        std::lock_guard<std::mutex> events_lock(events_mutex_);
        
        // Update vector clock with received event
        for (const auto& [node, clock] : event->vector_clock) {
            local_clock_[node] = std::max(local_clock_[node], clock);
        }
        
        // Increment own clock
        local_clock_[node_id_]++;
        
        // Store event
        events_[event->id] = event;
        
        // Add to time bucket
        add_to_time_bucket(event->timestamp, event->id);
    }
    
    // ========================================================================
    // CCRP Core: Causal Convergence in Recent Past
    // ========================================================================
    
    struct ConvergenceResult {
        bool converged;
        double confidence;
        std::chrono::steady_clock::time_point window_start;
        std::chrono::steady_clock::time_point window_end;
        std::vector<EventId> events_in_window;
    };
    
    ConvergenceResult check_causal_convergence() const {
        auto current_time = truetime_->now();
        auto window_start = current_time.earliest - recent_past_window_;
        auto window_end = current_time.earliest - recent_past_window_ / 2;
        
        std::lock_guard<std::mutex> events_lock(events_mutex_);
        std::lock_guard<std::mutex> conv_lock(convergence_mutex_);
        
        // Find events in the recent past window
        std::vector<EventId> events_in_window;
        for (const auto& [event_id, event] : events_) {
            if (event->timestamp >= window_start && event->timestamp <= window_end) {
                events_in_window.push_back(event_id);
            }
        }
        
        // Check causal ordering consistency
        bool causal_consistent = true;
        double confidence = 1.0;
        
        for (size_t i = 0; i < events_in_window.size() && causal_consistent; ++i) {
            for (size_t j = i + 1; j < events_in_window.size(); ++j) {
                auto event1 = events_.at(events_in_window[i]);
                auto event2 = events_.at(events_in_window[j]);
                
                if (event1->happens_before(*event2)) {
                    // Verify timestamp ordering matches causal ordering
                    if (event1->timestamp > event2->timestamp) {
                        causal_consistent = false;
                        confidence *= 0.8; // Reduce confidence for violations
                    }
                }
            }
        }
        
        // Adjust confidence based on time distance from present
        auto time_distance = std::chrono::duration_cast<std::chrono::microseconds>(
            current_time.earliest - window_end).count();
        confidence *= std::min(1.0, time_distance / 1000.0); // Increase confidence with age
        
        return ConvergenceResult{
            causal_consistent,
            confidence,
            window_start,
            window_end,
            events_in_window
        };
    }
    
    // ========================================================================
    // Spanner-inspired Commit Wait for CCRP
    // ========================================================================
    
    class CCRPTransaction {
    private:
        std::vector<std::shared_ptr<CausalEvent>> events_;
        std::chrono::steady_clock::time_point commit_timestamp_;
        CCRPNode* node_;
        
    public:
        CCRPTransaction(CCRPNode* node) : node_(node) {}
        
        void add_event(std::shared_ptr<CausalEvent> event) {
            events_.push_back(event);
        }
        
        // Spanner-style commit with causal wait
        bool commit() {
            if (events_.empty()) return false;
            
            // Determine commit timestamp (latest event timestamp)
            commit_timestamp_ = events_[0]->timestamp;
            for (const auto& event : events_) {
                commit_timestamp_ = std::max(commit_timestamp_, event->timestamp);
            }
            
            // Wait for causal dependencies to stabilize (CCRP extension)
            auto wait_time = commit_timestamp_ + node_->recent_past_window_;
            node_->truetime_->wait_until_past(wait_time);
            
            // Verify causal consistency before committing
            auto convergence = node_->check_causal_convergence();
            
            if (convergence.converged && convergence.confidence > 0.9) {
                // Commit all events atomically
                for (auto& event : events_) {
                    // Mark as committed with high confidence
                    event->data += " [COMMITTED]";
                }
                return true;
            }
            
            return false; // Abort if causal consistency not achieved
        }
    };
    
    std::unique_ptr<CCRPTransaction> begin_transaction() {
        return std::make_unique<CCRPTransaction>(this);
    }
    
    // ========================================================================
    // Temporal Query Interface
    // ========================================================================
    
    struct QueryResult {
        std::vector<std::shared_ptr<CausalEvent>> events;
        double consistency_confidence;
        std::chrono::steady_clock::time_point query_window_start;
        std::chrono::steady_clock::time_point query_window_end;
    };
    
    QueryResult query_events_in_recent_past(
        std::function<bool(const CausalEvent&)> predicate = nullptr) const {
        
        auto convergence = check_causal_convergence();
        QueryResult result;
        result.consistency_confidence = convergence.confidence;
        result.query_window_start = convergence.window_start;
        result.query_window_end = convergence.window_end;
        
        std::lock_guard<std::mutex> lock(events_mutex_);
        
        for (const auto& event_id : convergence.events_in_window) {
            auto event = events_.at(event_id);
            if (!predicate || predicate(*event)) {
                result.events.push_back(event);
            }
        }
        
        // Sort by causal order
        std::sort(result.events.begin(), result.events.end(),
                  [](const auto& a, const auto& b) {
                      return a->happens_before(*b);
                  });
        
        return result;
    }
    
    // ========================================================================
    // Configuration and Monitoring
    // ========================================================================
    
    void set_recent_past_window(std::chrono::microseconds window) {
        recent_past_window_ = window;
    }
    
    std::chrono::microseconds get_recent_past_window() const {
        return recent_past_window_;
    }
    
    size_t get_event_count() const {
        std::lock_guard<std::mutex> lock(events_mutex_);
        return events_.size();
    }
    
    void print_status() const {
        auto convergence = check_causal_convergence();
        std::cout << "Node " << node_id_ << " Status:\n"
                  << "  Events: " << get_event_count() << "\n"
                  << "  Causal Convergence: " << (convergence.converged ? "YES" : "NO") << "\n"
                  << "  Confidence: " << (convergence.confidence * 100) << "%\n"
                  << "  Recent Past Window: " << recent_past_window_.count() << "μs\n"
                  << "  Events in Window: " << convergence.events_in_window.size() << "\n";
    }

private:
    void add_to_time_bucket(std::chrono::steady_clock::time_point timestamp, 
                           EventId event_id) {
        std::lock_guard<std::mutex> lock(convergence_mutex_);
        time_buckets_[timestamp].push_back(event_id);
    }
};

// ============================================================================
// CCRP Distributed System Simulator
// ============================================================================

class CCRPSystem {
private:
    std::shared_ptr<TrueTimeAPI> truetime_;
    std::vector<std::unique_ptr<CCRPNode>> nodes_;
    std::mutex system_mutex_;
    
public:
    CCRPSystem(size_t num_nodes, int64_t base_uncertainty_us = 1000) 
        : truetime_(std::make_shared<TrueTimeAPI>()) {
        
        truetime_->set_uncertainty_us(base_uncertainty_us);
        
        for (size_t i = 0; i < num_nodes; ++i) {
            nodes_.push_back(std::make_unique<CCRPNode>(i, truetime_));
        }
    }
    
    CCRPNode* get_node(NodeId id) {
        if (id < nodes_.size()) {
            return nodes_[id].get();
        }
        return nullptr;
    }
    
    // Simulate network communication between nodes
    void broadcast_event(NodeId sender, std::shared_ptr<CausalEvent> event) {
        std::lock_guard<std::mutex> lock(system_mutex_);
        
        // Simulate network delay
        std::this_thread::sleep_for(std::chrono::microseconds(100));
        
        for (size_t i = 0; i < nodes_.size(); ++i) {
            if (i != sender) {
                nodes_[i]->receive_event(event);
            }
        }
    }
    
    void simulate_network_partition(std::chrono::milliseconds duration) {
        std::cout << "Simulating network partition for " 
                  << duration.count() << "ms\n";
        
        // Increase TrueTime uncertainty during partition
        truetime_->set_uncertainty_us(5000); // 5ms uncertainty
        
        std::this_thread::sleep_for(duration);
        
        // Restore normal uncertainty
        truetime_->set_uncertainty_us(1000); // 1ms uncertainty
        
        std::cout << "Network partition resolved\n";
    }
    
    void print_system_status() const {
        std::cout << "\n=== CCRP System Status ===\n";
        for (const auto& node : nodes_) {
            node->print_status();
            std::cout << "\n";
        }
    }
};

// ============================================================================
// Example Usage and Test Cases
// ============================================================================

void demonstrate_basic_ccrp() {
    std::cout << "=== Basic CCRP Demonstration ===\n";
    
    CCRPSystem system(3); // 3 nodes
    
    // Create some events
    auto node0 = system.get_node(0);
    auto node1 = system.get_node(1);
    auto node2 = system.get_node(2);
    
    // Node 0 creates an event
    auto event1 = node0->create_event("Initial state");
    system.broadcast_event(0, event1);
    
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    
    // Node 1 creates a dependent event
    auto event2 = node1->create_event("Update based on initial");
    system.broadcast_event(1, event2);
    
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    
    // Node 2 creates another event
    auto event3 = node2->create_event("Concurrent update");
    system.broadcast_event(2, event3);
    
    // Wait for recent past window to stabilize
    std::this_thread::sleep_for(std::chrono::milliseconds(15));
    
    system.print_system_status();
    
    // Query events in recent past
    auto query_result = node0->query_events_in_recent_past();
    std::cout << "Query result: " << query_result.events.size() 
              << " events with " << (query_result.consistency_confidence * 100) 
              << "% confidence\n";
}

void demonstrate_spanner_style_transactions() {
    std::cout << "\n=== Spanner-style CCRP Transactions ===\n";
    
    CCRPSystem system(2);
    auto node0 = system.get_node(0);
    
    // Begin transaction
    auto tx = node0->begin_transaction();
    
    // Add events to transaction
    auto event1 = node0->create_event("Account A: -$100");
    auto event2 = node0->create_event("Account B: +$100");
    
    tx->add_event(event1);
    tx->add_event(event2);
    
    // Commit with causal wait
    bool committed = tx->commit();
    
    std::cout << "Transaction " << (committed ? "COMMITTED" : "ABORTED") << "\n";
    
    system.print_system_status();
}

void demonstrate_network_partition_recovery() {
    std::cout << "\n=== Network Partition Recovery ===\n";
    
    CCRPSystem system(3);
    
    // Create events before partition
    auto node0 = system.get_node(0);
    auto event1 = node0->create_event("Before partition");
    system.broadcast_event(0, event1);
    
    // Simulate partition
    system.simulate_network_partition(std::chrono::milliseconds(20));
    
    // Create events during partition (won't propagate)
    auto event2 = node0->create_event("During partition");
    // Don't broadcast - simulating partition
    
    // Wait for recovery
    std::this_thread::sleep_for(std::chrono::milliseconds(15));
    
    // Now broadcast after partition
    system.broadcast_event(0, event2);
    
    // Wait for convergence
    std::this_thread::sleep_for(std::chrono::milliseconds(15));
    
    system.print_system_status();
}

// ============================================================================
// Main Function - Run Demonstrations
// ============================================================================

int main() {
    std::cout << "CCRP (Causal Convergence in Recent Past) Implementation Demo\n";
    std::cout << "============================================================\n";
    
    demonstrate_basic_ccrp();
    demonstrate_spanner_style_transactions();
    demonstrate_network_partition_recovery();
    
    std::cout << "\nDemo completed. Key CCRP concepts demonstrated:\n";
    std::cout << "1. Temporal bounds for consistency guarantees\n";
    std::cout << "2. Causal ordering preservation\n";
    std::cout << "3. Probabilistic confidence intervals\n";
    std::cout << "4. TrueTime-inspired uncertainty handling\n";
    std::cout << "5. Spanner-style commit wait adapted for causal convergence\n";
    
    return 0;
}
