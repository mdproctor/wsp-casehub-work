---
layout: post
title: "The Builder That Grew"
date: 2026-11-17
entry_type: note
subtype: diary
projects: [casehubio/connectors]
tags: [java, api-design, teams, builder-pattern, defensive-programming]
---

# The Builder That Grew

Teams cards started as raw JSON strings. Hardcoded constants like this were the reference implementation:

```java
private static final String CARD = "{\"type\":\"AdaptiveCard\",\"body\":[" +
    "{\"type\":\"TextBlock\",\"text\":\"Link Preview\",\"weight\":\"bolder\"}," +
    "{\"type\":\"TextBlock\",\"text\":\"This is a preview.\"}]," +
    "\"actions\":[{\"type\":\"Action.OpenUrl\",\"title\":\"View\"," +
    "\"url\":\"https://example.com\"}]}";
```

It works. But the moment you need a second card, you're copying strings. The moment you need a dynamic title, you're concatenating JSON by hand. The moment a field is optional, you're maintaining two versions of every constant.

## Why builder, not template

The obvious alternative is a JSON template engine — write a skeleton, fill placeholders. I rejected this for the same reason I reject string templates in SQL: the structure of the output is invisible at the call site. You read a method call and have no idea what JSON shape it produces without tracing through the template.

A fluent builder makes the shape visible:

```java
ComposeAttachment.hero("Link Preview")
                 .text("This is a preview of the linked content.")
                 .button("openUrl", "View", "https://example.com")
                 .build();
```

Every field is named. Optional fields are absent, not null-checked in a template. The build site reads like a declaration of what the card contains.

## The shared helper that made it work

The interesting design decision was `cardContent()` — a private static method that both `Builder` (adaptive cards) and `CardBuilder` (hero/thumbnail) call to emit the common JSON fields. Title, subtitle, text, images — these appear in every card type, in the same order. The alternative was duplicating the emission logic in each builder's `build()` method.

![ComposeAttachment builder layers](svg/2026-11-17-card-builder-layers.svg)

Adding `text()` to `CardBuilder` meant adding one parameter to `cardContent()` and one line of emission. The adaptive `Builder` passes `null` for text since previews don't use it. Same pattern for subtitle, same pattern for imageUrl — the helper absorbs optionality through nulls, and callers only expose what their card type supports.

This is the kind of factoring that pays off silently. Each new card property costs one field, one setter, one parameter, one null check — not a cross-cutting edit across two independent builders.

## The record that shares too much

A side project during this work: `InvokeRequest` and `InboundMessage` both receive the same metadata map from `TeamsInboundConnector`. Same reference, same backing `HashMap`. Right now all handlers are read-only, so nothing breaks. But a handler that calls `request.metadata().put("key", "value")` would mutate the `InboundMessage`'s metadata too — across a CDI event boundary, silently.

The fix is a compact constructor on the record:

```java
public record InvokeRequest( /* fields */ ) {
    public InvokeRequest {
        metadata = Map.copyOf(metadata);
    }
}
```

`Map.copyOf()` on an already-unmodifiable map (like `Map.of()`) returns the same instance — zero allocation. On a mutable `HashMap`, it creates an unmodifiable snapshot. The cost is nothing in production and the safety is permanent.

What's worth noting: `InboundMessage` has the same vulnerability. It already does `List.copyOf(attachments)` in its compact constructor but not `Map.copyOf(metadata)`. The asymmetry is a gap, not a design choice — fixing it is filed.

## The dog-food test

I migrated `RefLinkUnfurlHandler` from a raw AdaptiveCard constant to `hero().text().button()`. The original was 160 characters of escaped JSON. The replacement is four chained method calls. Same output, readable at a glance, and it exercises every new API method in production-shaped code.

The other two ref handlers — `RefSearchHandler` and `RefFetchTaskHandler` — stayed on `adaptive()`. The search handler genuinely needs AdaptiveCard format for its content plus a thumbnail preview. The fetch handler uses `Input.Text` and `Action.Submit`, which are AdaptiveCard-only features. Builder migration is for the cases where hero/thumbnail is the right card type. Forcing everything through one API shape is worse than the JSON constants.

The builder API now covers `title`, `subtitle`, `text`, `imageUrl`, `buttons`, `tap`, and `preview` — enough to express most Teams card scenarios without touching JSON directly. The gap is `preview(title, subtitle, imageUrl)` on `CardBuilder`, which the adaptive `Builder` already supports. Filed for follow-up.
