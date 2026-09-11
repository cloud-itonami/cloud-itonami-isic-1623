(ns woodcontainer.governor
  "Wooden Container Shop Plant Operations Governor -- the independent
  compliance layer that earns the WoodenContainerAdvisor the right to
  commit. The advisor has no notion of whether a piece of equipment
  it wants to schedule maintenance against has actually been
  inspected/registered, whether a batch it wants to coordinate a
  shipment against has actually been QC-verified/registered, whether
  a maintenance proposal secretly tries to FINALIZE (rather than
  merely draft-schedule) a cutting/assembly-line run, whether a
  shipment proposal's own claimed unit count would blow through the
  batch's own logged production unit count, or when an act stops
  being a coordination proposal and becomes direct cutting/assembly-
  line-equipment control, so this MUST be a separate system able to
  *reject* a proposal and fall back to HOLD.

  `:itonami.blueprint/governor` is `:wooden-container-shop-plant-
  operations-governor` (see docs/adr/0001-architecture.md).

  Checks below, ALL HARD violations except the confidence/high-stakes
  gate (SOFT -- asks a human to look, and the human may approve):

    1. Request-level propose-only  -- did the CALLER's own request
                                       actually declare `:effect
                                       :propose`? Any other value is a
                                       mis-wired/compromised caller
                                       trying to bypass proposal-only
                                       mode -- HARD, unconditional,
                                       evaluated BEFORE anything else.
    2. Closed op allowlist         -- is `:op` one of the four ops this
                                       actor is authorized to coordinate?
                                       Anything else -- HARD hold.
    3. Closed effect allowlist     -- is the PROPOSAL's own `:effect`
                                       (what would actually commit) one
                                       of the four propose-shaped
                                       effects? A proposal effect
                                       outside this set (e.g. a
                                       hallucinated `:router/actuate` or
                                       `:cutting-assembly-line/finalize`)
                                       is the 'direct cutting/assembly-
                                       line-equipment control' scope
                                       violation this actor must NEVER
                                       perform -- HARD, PERMANENT,
                                       unconditional.
    4. Cutting/assembly-line-
       finalize blocked            -- for `:schedule-maintenance`, does
                                       the proposal's own `:value`
                                       declare `:finalize? true`?
                                       Finalizing a cutting/assembly-
                                       line run is this actor's other
                                       permanent scope boundary (see
                                       README `What this actor does
                                       NOT do`) -- HARD, PERMANENT,
                                       unconditional. NO phase and NO
                                       human approval can ever override
                                       this (see `woodcontainer.phase`:
                                       this op is never a member of any
                                       phase's `:auto` set either --
                                       two independent layers agree).
    5. Equipment not verified/
       registered                  -- for `:schedule-maintenance`,
                                       INDEPENDENTLY verify the
                                       referenced equipment's own
                                       `:verified?` AND `:registered?`
                                       are both true
                                       (`woodcontainer.registry/
                                       equipment-ready?`) -- never
                                       trust the advisor's own rationale
                                       about verification/registration
                                       status. Grounded in this
                                       blueprint's own HARD invariant
                                       ('shop/batch record must be
                                       independently verified/
                                       registered before any action'):
                                       maintenance must never be
                                       scheduled against equipment
                                       whose own conditions have not
                                       actually been inspected or
                                       whose registration is not
                                       actually on file.
    6. Already scheduled           -- for `:schedule-maintenance`,
                                       refuses to schedule the SAME
                                       maintenance record twice, off a
                                       dedicated `:scheduled?` fact
                                       (never a `:status` value).
    7. Batch not verified/
       registered                  -- for `:coordinate-shipment`,
                                       INDEPENDENTLY verify the
                                       referenced batch's own
                                       `:verified?` AND `:registered?`
                                       are both true
                                       (`woodcontainer.registry/batch-
                                       ready?`) -- never trust the
                                       advisor's own rationale. Also
                                       part of the 'shop/batch record'
                                       HARD invariant: a batch's own
                                       verified/registered status is as
                                       much a ground-truth fact as an
                                       equipment unit's own.
    8. Shipment unit count
       exceeded                     -- for `:coordinate-shipment`,
                                       INDEPENDENTLY recompute whether
                                       the batch's own recorded
                                       `:shipped-unit-count` plus the
                                       proposal's own claimed
                                       `:unit-count` would exceed the
                                       batch's own recorded
                                       `:unit-count`
                                       (`woodcontainer.registry/
                                       shipment-unit-count-exceeded?`)
                                       -- ground truth from the
                                       batch's own permanent fields,
                                       never a self-reported unit-count
                                       claim.
    9. Invalid dimensional spec    -- for `:log-production-batch`, if
                                       the patch declares a
                                       `:dimensional-spec` outside the
                                       closed known set
                                       (`woodcontainer.registry/
                                       dimensional-spec-valid?`), the
                                       batch record is rejected rather
                                       than let a fabricated
                                       dimensional spec through.
   10. Invalid output-quality      -- for `:log-production-batch`, if
                                       the patch declares an
                                       `:output-quality-percent` that
                                       is not a physically plausible
                                       reading
                                       (`woodcontainer.registry/
                                       output-quality-valid?`), the
                                       batch record is rejected rather
                                       than let fabricated/sensor-error
                                       data through.
   11. Confidence floor / high-
       stakes gate                  -- LLM confidence below threshold,
                                       OR the proposal's own `:stake` is
                                       in `high-stakes`
                                       (`:coordination/safety-concern`,
                                       ALWAYS set for `:flag-safety-
                                       concern` -- materials-safety,
                                       equipment-safety, or ISPM-15
                                       heat-treatment-compliance) --
                                       escalate to a human plant
                                       supervisor. SOFT: the human may
                                       approve."
  (:require [woodcontainer.registry :as registry]
            [woodcontainer.store :as store]))

(def confidence-floor 0.6)

(def allowed-ops
  "The closed allowlist of coordination proposals this actor may ever
  route -- see README `What this actor does`."
  #{:log-production-batch :schedule-maintenance
    :flag-safety-concern :coordinate-shipment})

(def allowed-proposal-effects
  "The closed allowlist of SSoT-mutation effects a proposal may declare
  -- all four are propose-shaped drafts, NEVER a direct cutting/
  assembly-line-equipment-control or cutting/assembly-line-finalize
  effect."
  #{:batch/upsert :maintenance/schedule
    :safety-concern/flag :shipment/propose})

(def high-stakes
  "Stakes grave enough to always require a human, even when clean.
  Safety concerns are the one op in this domain that always demands
  human eyes regardless of confidence."
  #{:coordination/safety-concern})

;; ----------------------------- checks -----------------------------

(defn- no-propose-effect-violations
  "HARD, unconditional, evaluated first: the caller's own request MUST
  declare `:effect :propose` -- any other value is a mis-wired or
  compromised caller trying to bypass proposal-only mode."
  [{:keys [effect]}]
  (when (not= effect :propose)
    [{:rule :not-propose-effect
      :detail (str "request :effect は :propose のみ許可 (受信値: " (pr-str effect) ")")}]))

(defn- unknown-op-violations
  "HARD: `:op` must be one of the closed allowlist this actor
  coordinates -- never route an unrecognized operation."
  [{:keys [op]}]
  (when-not (contains? allowed-ops op)
    [{:rule :unknown-op
      :detail (str op " はこの actor が扱う操作の許可リストに無い")}]))

(defn- equipment-control-blocked-violations
  "HARD, PERMANENT: the proposal's own `:effect` -- what would actually
  commit -- must be within the closed propose-shaped effect allowlist.
  Anything else (direct cutting/assembly-line-equipment control, a
  fabricated actuation effect) is this actor's central scope
  boundary."
  [proposal]
  (when-not (contains? allowed-proposal-effects (:effect proposal))
    [{:rule :equipment-control-blocked
      :detail (str "proposal :effect (" (pr-str (:effect proposal))
                   ") は切断・組立ラインの設備の直接操作/稼働確定に該当する可能性があり、恒久的に禁止")}]))

(defn- cutting-assembly-line-finalize-blocked-violations
  "HARD, PERMANENT, unconditional: a `:schedule-maintenance` proposal
  whose own `:value` declares `:finalize? true` is attempting to
  finalize a cutting/assembly-line run directly -- this actor may
  only ever propose/schedule a DRAFT maintenance window, never
  finalize/execute a cutting/assembly-line run. No override, ever."
  [{:keys [op]} proposal]
  (when (and (= op :schedule-maintenance)
             (true? (:finalize? (:value proposal))))
    [{:rule :cutting-assembly-line-finalize-blocked
      :detail "切断・組立ラインの稼働確定(finalize)提案は恒久的に禁止 -- 提案(draft)のみ許可"}]))

(defn- equipment-not-verified-violations
  "For `:schedule-maintenance`, INDEPENDENTLY verify the referenced
  equipment exists and is both `:verified?` AND `:registered?` --
  never trust the advisor's own report. This is the HARD invariant
  ('shop/batch record must be independently verified/registered
  before any action')."
  [{:keys [op]} proposal st]
  (when (= op :schedule-maintenance)
    (let [equipment-id (:equipment-id (:value proposal))
          eq (and equipment-id (store/equipment-unit st equipment-id))]
      (when-not (and eq (registry/equipment-ready? eq))
        [{:rule :equipment-not-verified
          :detail (str equipment-id " は未検証または未登録、もしくは存在しない -- 検証済み・登録済み設備記録が無い状態での保守作業予定提案")}]))))

(defn- already-scheduled-violations
  "For `:schedule-maintenance`, refuses to schedule the SAME
  maintenance record twice, off a dedicated `:scheduled?` fact (never
  a `:status` value)."
  [{:keys [op subject]} st]
  (when (= op :schedule-maintenance)
    (when (store/maintenance-already-scheduled? st subject)
      [{:rule :already-scheduled
        :detail (str subject " は既にスケジュール済み")}])))

(defn- batch-not-verified-violations
  "For `:coordinate-shipment`, INDEPENDENTLY verify the referenced
  batch exists and is both `:verified?` AND `:registered?` -- never
  trust the advisor's own report. Also part of the 'shop/batch record
  must be independently verified/registered before any action' HARD
  invariant."
  [{:keys [op]} proposal st]
  (when (= op :coordinate-shipment)
    (let [batch-id (:batch-id (:value proposal))
          b (and batch-id (store/batch st batch-id))]
      (when-not (and b (registry/batch-ready? b))
        [{:rule :batch-not-verified
          :detail (str batch-id " は未検証または未登録、もしくは存在しない -- 検証済み・登録済みバッチ記録が無い状態での出荷調整提案")}]))))

(defn- shipment-unit-count-exceeded-violations
  "For `:coordinate-shipment`, INDEPENDENTLY recompute whether the
  batch's own recorded shipped-to-date unit count plus the proposal's
  own claimed unit count would exceed the batch's own recorded
  `:unit-count` -- ground truth from the batch's own permanent
  fields, never a self-reported unit-count claim."
  [{:keys [op]} proposal st]
  (when (= op :coordinate-shipment)
    (let [{:keys [batch-id unit-count]} (:value proposal)
          b (and batch-id (store/batch st batch-id))]
      (cond
        ;; No batch, no recorded capacity, or no stated amount: the headroom
        ;; cannot be computed, so it is not headroom. This used to fall
        ;; through as "not over capacity" and ship.
        (not (registry/shipment-unit-count-exceeded-checkable? b unit-count))
        [{:rule :shipment-unit-count-exceeded
          :detail "生産量/既存出荷実績/申請量のいずれかが数値として確定できない -- 空き容量を検算できないため出荷しない"}]

        (registry/shipment-unit-count-exceeded? b unit-count)
        [{:rule :shipment-unit-count-exceeded
          :detail (str batch-id " の記録済み生産数(" (:unit-count b)
                       "個)を、既存出荷実績(" (:shipped-unit-count b 0)
                       "個)+今回申請(" unit-count "個)が超過")}]))))

(defn- invalid-dimensional-spec-violations
  "For `:log-production-batch`, if the patch declares a
  `:dimensional-spec` outside the closed known set, reject rather
  than let a fabricated dimensional spec through."
  [{:keys [op]} proposal]
  (when (= op :log-production-batch)
    (let [spec (:dimensional-spec (:value proposal))]
      (when (and (some? spec) (not (registry/dimensional-spec-valid? spec)))
        [{:rule :invalid-dimensional-spec
          :detail (str spec " は既知の dimensional-spec 値ではない")}]))))

(defn- invalid-output-quality-violations
  "For `:log-production-batch`, if the patch declares an
  `:output-quality-percent` that is not a physically plausible
  reading, reject rather than let fabricated/sensor-error data
  through."
  [{:keys [op]} proposal]
  (when (= op :log-production-batch)
    (let [oq (:output-quality-percent (:value proposal))]
      (when (and (some? oq) (not (registry/output-quality-valid? oq)))
        [{:rule :invalid-output-quality
          :detail (str oq "% は物理的に妥当な出力品質率の範囲外")}]))))

(defn check
  "Censors a WoodenContainerAdvisor proposal against the governor
  rules. Returns {:ok? bool :violations [..] :confidence c :escalate?
  bool :high-stakes? bool :hard? bool}."
  [request _context proposal st]
  (let [hard (into []
                   (concat (no-propose-effect-violations request)
                           (unknown-op-violations request)
                           (equipment-control-blocked-violations proposal)
                           (cutting-assembly-line-finalize-blocked-violations request proposal)
                           (equipment-not-verified-violations request proposal st)
                           (already-scheduled-violations request st)
                           (batch-not-verified-violations request proposal st)
                           (shipment-unit-count-exceeded-violations request proposal st)
                           (invalid-dimensional-spec-violations request proposal)
                           (invalid-output-quality-violations request proposal)))
        conf (:confidence proposal 0.0)
        low? (< conf confidence-floor)
        stakes? (boolean (high-stakes (:stake proposal)))
        hard? (boolean (seq hard))]
    {:ok?          (and (not hard?) (not low?) (not stakes?))
     :violations   hard
     :confidence   conf
     :hard?        hard?
     :escalate?    (and (not hard?) (or low? stakes?))
     :high-stakes? stakes?}))

(defn hold-fact
  "The audit fact written when a proposal is rejected (HOLD)."
  [request context verdict]
  {:t          :governor-hold
   :op         (:op request)
   :actor      (:actor-id context)
   :subject    (:subject request)
   :disposition :hold
   :basis      (mapv :rule (:violations verdict))
   :violations (:violations verdict)
   :confidence (:confidence verdict)})
