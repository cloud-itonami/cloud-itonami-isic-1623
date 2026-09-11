(ns woodcontainer.sim
  "Demo driver -- `clojure -M:dev:run`. Walks a clean shop through
  intake -> maintenance scheduling (escalate/approve) -> safety-concern
  flag (escalate/approve) -> shipment coordination (escalate/approve),
  then shows HARD-hold scenarios: a mis-wired request whose own
  `:effect` is not `:propose`, an unrecognized op, maintenance
  scheduled against an UNVERIFIED/unregistered equipment unit, a
  shipment coordinated against an UNVERIFIED/unregistered batch, a
  shipment proposal that would exceed the batch's own logged
  production unit count, a proposal that tries to FINALIZE a cutting/
  assembly-line run (permanently blocked, no override), a
  double-schedule of the same maintenance window, a production-batch
  patch with a fabricated dimensional spec, and a production-batch
  patch with an implausible output-quality reading.

  Like every sibling actor's own demo, each check is exercised directly
  and independently below, one request per HARD-hold scenario, the SAME
  'exercise the failure mode directly, never only via a happy-path
  actuation' discipline `parksafety`'s ADR-2607071922 Decision 5 and
  every sibling since establish."
  (:require [langgraph.graph :as g]
            [woodcontainer.store :as store]
            [woodcontainer.operation :as op]))

(def coordinator {:actor-id "coord-1" :actor-role :plant-coordinator :phase 3})

(defn- exec-op [actor tid request context]
  (g/run* actor {:request request :context context} {:thread-id tid}))

(defn- approve! [actor tid]
  (g/run* actor {:approval {:status :approved :by "coord-1"}} {:thread-id tid :resume? true}))

(defn -main [& _args]
  (let [db (-> (store/mem-store) (store/sample-data!))
        actor (op/build db)]

    (println "== log-production-batch batch-001 (clean patch -> phase-3 auto-commit) ==")
    (println (exec-op actor "t1"
                       {:op :log-production-batch :effect :propose :subject "batch-001"
                        :patch {:dimensional-spec :pallet-block-style :last-assessed "2026-07-14"}}
                       coordinator))

    (println "== schedule-maintenance mnt-1 on equip-001 (verified, registered, crate/pallet nailer -- escalates, approve) ==")
    (let [r (exec-op actor "t2"
                      {:op :schedule-maintenance :effect :propose :subject "mnt-1"
                       :value {:equipment-id "equip-001" :maintenance-type :nailer-blade-and-bit-service
                               :scheduled-date "2026-08-01" :finalize? false}}
                      coordinator)]
      (println r)
      (println "-- human plant supervisor approves --")
      (println (approve! actor "t2")))

    (println "== flag-safety-concern concern-1 on equip-001 (always escalates -- approve) ==")
    (let [r (exec-op actor "t3"
                      {:op :flag-safety-concern :effect :propose :subject "concern-1"
                       :value {:equipment-id "equip-001" :severity :moderate
                               :description "ISPM-15熱処理証明未更新の疑い (heat-treatment-compliance concern)"}}
                      coordinator)]
      (println r)
      (println "-- human plant supervisor approves --")
      (println (approve! actor "t3")))

    (println "== coordinate-shipment ship-1 on batch-001 (verified, registered, within unit count -- escalates, approve) ==")
    (let [r (exec-op actor "t4"
                      {:op :coordinate-shipment :effect :propose :subject "ship-1"
                       :value {:batch-id "batch-001" :unit-count 50
                               :destination "buyer-site-north"}}
                      coordinator)]
      (println r)
      (println "-- human shipping approver approves --")
      (println (approve! actor "t4")))

    (println "\n-- HARD-hold scenarios --\n")

    (println "== log-production-batch with :effect other than :propose -> HARD hold (structural) ==")
    (println (exec-op actor "t5"
                       {:op :log-production-batch :effect :direct-write :subject "batch-001"
                        :patch {:dimensional-spec :pallet-block-style}}
                       coordinator))

    (println "== unrecognized op -> HARD hold ==")
    (println (exec-op actor "t6"
                       {:op :actuate-nailer :effect :propose :subject "batch-001"}
                       coordinator))

    (println "== schedule-maintenance mnt-2 on equip-002 (UNVERIFIED/unregistered stave jointer -> HARD hold) ==")
    (println (exec-op actor "t7"
                       {:op :schedule-maintenance :effect :propose :subject "mnt-2"
                        :value {:equipment-id "equip-002" :maintenance-type :stave-jointer-blade-calibration
                                :scheduled-date "2026-08-01" :finalize? false}}
                       coordinator))

    (println "== coordinate-shipment ship-2 on batch-003 (UNVERIFIED/unregistered batch -> HARD hold) ==")
    (println (exec-op actor "t8"
                       {:op :coordinate-shipment :effect :propose :subject "ship-2"
                        :value {:batch-id "batch-003" :unit-count 10
                                :destination "buyer-site-south"}}
                       coordinator))

    (println "== coordinate-shipment ship-3 on batch-002 (10 units would exceed unit-count 80 vs shipped 75 -> HARD hold) ==")
    (println (exec-op actor "t9"
                       {:op :coordinate-shipment :effect :propose :subject "ship-3"
                        :value {:batch-id "batch-002" :unit-count 10
                                :destination "buyer-site-east"}}
                       coordinator))

    (println "== schedule-maintenance mnt-3 on equip-001 with :finalize? true -> HARD hold, PERMANENT, never reaches a human ==")
    (println (exec-op actor "t10"
                       {:op :schedule-maintenance :effect :propose :subject "mnt-3"
                        :value {:equipment-id "equip-001" :maintenance-type :cutting-assembly-line-run-finalize
                                :scheduled-date "2026-09-01" :finalize? true}}
                       coordinator))

    (println "== schedule-maintenance mnt-1 AGAIN (double-schedule -> HARD hold) ==")
    (println (exec-op actor "t11"
                       {:op :schedule-maintenance :effect :propose :subject "mnt-1"
                        :value {:equipment-id "equip-001" :maintenance-type :nailer-blade-and-bit-service
                                :scheduled-date "2026-08-01" :finalize? false}}
                       coordinator))

    (println "== log-production-batch batch-001 with a fabricated dimensional spec -> HARD hold ==")
    (println (exec-op actor "t12"
                       {:op :log-production-batch :effect :propose :subject "batch-001"
                        :patch {:dimensional-spec :custom-oversize-shipping-cage}}
                       coordinator))

    (println "== log-production-batch batch-001 with an implausible output-quality reading -> HARD hold ==")
    (println (exec-op actor "t13"
                       {:op :log-production-batch :effect :propose :subject "batch-001"
                        :patch {:output-quality-percent 999.0}}
                       coordinator))

    (println "\n== audit ledger ==")
    (doseq [f (store/ledger db)] (println f))

    (println "\n== draft maintenance records ==")
    (doseq [r (store/maintenance-history db)] (println r))

    (println "\n== draft shipment records ==")
    (doseq [r (store/shipment-history db)] (println r))))
