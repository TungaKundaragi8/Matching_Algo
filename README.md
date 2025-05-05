package com.example.reconciliation.model;

import java.util.List;

public class ReconciliationResult {
    private List<Record> matched;
    private List<Record> unmatched;
    private List<Record> excluded;
    private int matchedCount;
    private int unmatchedCount;
    private int excludedCount;

    public ReconciliationResult(List<Record> matched, List<Record> unmatched, List<Record> excluded) {
        this.matched = matched;
        this.unmatched = unmatched;
        this.excluded = excluded;
        this.matchedCount = matched.size();
        this.unmatchedCount = unmatched.size();
        this.excludedCount = excluded.size();
    }

    public List<Record> getMatched() {
        return matched;
    }

    public void setMatched(List<Record> matched) {
        this.matched = matched;
        this.matchedCount = matched.size();
    }

    public List<Record> getUnmatched() {
        return unmatched;
    }

    public void setUnmatched(List<Record> unmatched) {
        this.unmatched = unmatched;
        this.unmatchedCount = unmatched.size();
    }

    public List<Record> getExcluded() {
        return excluded;
    }

    public void setExcluded(List<Record> excluded) {
        this.excluded = excluded;
        this.excludedCount = excluded.size();
    }

    public int getMatchedCount() {
        return matchedCount;
    }

    public int getUnmatchedCount() {
        return unmatchedCount;
    }

    public int getExcludedCount() {
        return excludedCount;
    }
}
