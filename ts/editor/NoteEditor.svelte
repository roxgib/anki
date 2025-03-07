<!--
Copyright: Ankitects Pty Ltd and contributors
License: GNU AGPL, version 3 or later; http://www.gnu.org/licenses/agpl.html
-->
<script context="module" lang="ts">
    import type { Writable } from "svelte/store";

    import Collapsible from "../components/Collapsible.svelte";
    import type { EditingInputAPI } from "./EditingArea.svelte";
    import type { EditorToolbarAPI } from "./editor-toolbar";
    import type { EditorFieldAPI } from "./EditorField.svelte";
    import FieldState from "./FieldState.svelte";
    import LabelContainer from "./LabelContainer.svelte";
    import LabelName from "./LabelName.svelte";

    export interface NoteEditorAPI {
        fields: EditorFieldAPI[];
        hoveredField: Writable<EditorFieldAPI | null>;
        focusedField: Writable<EditorFieldAPI | null>;
        focusedInput: Writable<EditingInputAPI | null>;
        toolbar: EditorToolbarAPI;
    }

    import { registerPackage } from "../lib/runtime-require";
    import contextProperty from "../sveltelib/context-property";
    import lifecycleHooks from "../sveltelib/lifecycle-hooks";

    const key = Symbol("noteEditor");
    const [context, setContextProperty] = contextProperty<NoteEditorAPI>(key);
    const [lifecycle, instances, setupLifecycleHooks] = lifecycleHooks<NoteEditorAPI>();

    export { context };

    registerPackage("anki/NoteEditor", {
        context,
        lifecycle,
        instances,
    });
</script>

<script lang="ts">
    import { onMount, tick } from "svelte";
    import { get, writable } from "svelte/store";

    import Absolute from "../components/Absolute.svelte";
    import Badge from "../components/Badge.svelte";
    import StickyContainer from "../components/StickyContainer.svelte";
    import { bridgeCommand } from "../lib/bridgecommand";
    import { TagEditor } from "../tag-editor";
    import { ChangeTimer } from "./change-timer";
    import DecoratedElements from "./DecoratedElements.svelte";
    import { clearableArray } from "./destroyable";
    import DuplicateLink from "./DuplicateLink.svelte";
    import EditorToolbar from "./editor-toolbar";
    import type { FieldData } from "./EditorField.svelte";
    import EditorField from "./EditorField.svelte";
    import FieldDescription from "./FieldDescription.svelte";
    import Fields from "./Fields.svelte";
    import FieldsEditor from "./FieldsEditor.svelte";
    import FrameElement from "./FrameElement.svelte";
    import { alertIcon } from "./icons";
    import ImageHandle from "./image-overlay";
    import MathjaxHandle from "./mathjax-overlay";
    import MathjaxElement from "./MathjaxElement.svelte";
    import Notification from "./Notification.svelte";
    import PlainTextInput from "./plain-text-input";
    import PlainTextBadge from "./PlainTextBadge.svelte";
    import RichTextInput, { editingInputIsRichText } from "./rich-text-input";
    import RichTextBadge from "./RichTextBadge.svelte";

    function quoteFontFamily(fontFamily: string): string {
        // generic families (e.g. sans-serif) must not be quoted
        if (!/^[-a-z]+$/.test(fontFamily)) {
            fontFamily = `"${fontFamily}"`;
        }
        return fontFamily;
    }

    const size = 1.6;
    const wrap = true;

    const fieldStores: Writable<string>[] = [];
    let fieldNames: string[] = [];
    export function setFields(fs: [string, string][]): void {
        // this is a bit of a mess -- when moving to Rust calls, we should make
        // sure to have two backend endpoints for:
        // * the note, which can be set through this view
        // * the fieldname, font, etc., which cannot be set

        const newFieldNames: string[] = [];

        for (const [index, [fieldName]] of fs.entries()) {
            newFieldNames[index] = fieldName;
        }

        for (let i = fieldStores.length; i < newFieldNames.length; i++) {
            const newStore = writable("");
            fieldStores[i] = newStore;
            newStore.subscribe((value) => updateField(i, value));
        }

        for (
            let i = fieldStores.length;
            i > newFieldNames.length;
            i = fieldStores.length
        ) {
            fieldStores.pop();
        }

        for (const [index, [, fieldContent]] of fs.entries()) {
            fieldStores[index].set(fieldContent);
        }

        fieldNames = newFieldNames;
    }

    let fieldsCollapsed: boolean[] = [];
    export function setCollapsed(fs: boolean[]): void {
        fieldsCollapsed = fs;
    }

    let richTextsHidden: boolean[] = [];
    let plainTextsHidden: boolean[] = [];
    let plainTextDefaults: boolean[] = [];

    export function setPlainTexts(fs: boolean[]): void {
        richTextsHidden = fs;
        plainTextsHidden = Array.from(fs, (v) => !v);
        plainTextDefaults = [...richTextsHidden];
    }

    function setMathjaxEnabled(enabled: boolean): void {
        mathjaxConfig.enabled = enabled;
    }

    let fieldDescriptions: string[] = [];
    export function setDescriptions(fs: string[]): void {
        fieldDescriptions = fs;
    }

    let fonts: [string, number, boolean][] = [];

    const fields = clearableArray<EditorFieldAPI>();

    export function setFonts(fs: [string, number, boolean][]): void {
        fonts = fs;
    }

    export function focusField(index: number | null): void {
        tick().then(() => {
            if (typeof index === "number") {
                if (!(index in fields)) {
                    return;
                }

                fields[index].editingArea?.refocus();
            } else {
                $focusedInput?.refocus();
            }
        });
    }

    const tags = writable<string[]>([]);
    export function setTags(ts: string[]): void {
        $tags = ts;
    }

    let noteId: number | null = null;
    export function setNoteId(ntid: number): void {
        // TODO this is a hack, because it requires the NoteEditor to know implementation details of the PlainTextInput.
        // It should be refactored once we work on our own Undo stack
        for (const pi of plainTextInputs) {
            pi.api.codeMirror.editor.then((editor) => editor.clearHistory());
        }
        noteId = ntid;
    }

    function getNoteId(): number | null {
        return noteId;
    }

    let cols: ("dupe" | "")[] = [];
    export function setBackgrounds(cls: ("dupe" | "")[]): void {
        cols = cls;
    }

    let hint: string = "";
    export function setClozeHint(hnt: string): void {
        hint = hnt;
    }

    $: fieldsData = fieldNames.map((name, index) => ({
        name,
        plainText: plainTextDefaults[index],
        description: fieldDescriptions[index],
        fontFamily: quoteFontFamily(fonts[index][0]),
        fontSize: fonts[index][1],
        direction: fonts[index][2] ? "rtl" : "ltr",
        collapsed: fieldsCollapsed[index],
    })) as FieldData[];

    function saveTags({ detail }: CustomEvent): void {
        bridgeCommand(`saveTags:${JSON.stringify(detail.tags)}`);
    }

    const fieldSave = new ChangeTimer();

    function updateField(index: number, content: string): void {
        fieldSave.schedule(
            () => bridgeCommand(`key:${index}:${getNoteId()}:${content}`),
            600,
        );
    }

    export function saveFieldNow(): void {
        /* this will always be a key save */
        fieldSave.fireImmediately();
    }

    export function saveOnPageHide() {
        if (document.visibilityState === "hidden") {
            // will fire on session close and minimize
            saveFieldNow();
        }
    }

    export function focusIfField(x: number, y: number): boolean {
        const elements = document.elementsFromPoint(x, y);
        const first = elements[0];

        if (first.shadowRoot) {
            const richTextInput = first.shadowRoot.lastElementChild! as HTMLElement;
            richTextInput.focus();
            return true;
        }

        return false;
    }

    let richTextInputs: RichTextInput[] = [];
    $: richTextInputs = richTextInputs.filter(Boolean);

    let plainTextInputs: PlainTextInput[] = [];
    $: plainTextInputs = plainTextInputs.filter(Boolean);

    const toolbar: Partial<EditorToolbarAPI> = {};

    import { mathjaxConfig } from "../editable/mathjax-element";
    import { wrapInternal } from "../lib/wrap";
    import { refocusInput } from "./helpers";
    import * as oldEditorAdapter from "./old-editor-adapter";

    onMount(() => {
        function wrap(before: string, after: string): void {
            if (!$focusedInput || !editingInputIsRichText($focusedInput)) {
                return;
            }

            $focusedInput.element.then((element) => {
                wrapInternal(element, before, after, false);
            });
        }

        Object.assign(globalThis, {
            setFields,
            setCollapsed,
            setPlainTexts,
            setDescriptions,
            setFonts,
            focusField,
            setTags,
            setBackgrounds,
            setClozeHint,
            saveNow: saveFieldNow,
            focusIfField,
            getNoteId,
            setNoteId,
            wrap,
            setMathjaxEnabled,
            ...oldEditorAdapter,
        });

        document.addEventListener("visibilitychange", saveOnPageHide);
        return () => document.removeEventListener("visibilitychange", saveOnPageHide);
    });

    let apiPartial: Partial<NoteEditorAPI> = {};
    export { apiPartial as api };

    const hoveredField: NoteEditorAPI["hoveredField"] = writable(null);
    const focusedField: NoteEditorAPI["focusedField"] = writable(null);
    const focusedInput: NoteEditorAPI["focusedInput"] = writable(null);

    const api: NoteEditorAPI = {
        ...apiPartial,
        hoveredField,
        focusedField,
        focusedInput,
        toolbar: toolbar as EditorToolbarAPI,
        fields,
    };

    setContextProperty(api);
    setupLifecycleHooks(api);
</script>

<!--
@component
Serves as a pre-slotted convenience component which combines all the common
components and functionality for general note editing.

Functionality exclusive to specifc note-editing views (e.g. in the browser or
the AddCards dialog) should be implemented in the user of this component.
-->
<div class="note-editor">
    <FieldsEditor>
        <EditorToolbar {size} {wrap} api={toolbar}>
            <slot slot="notetypeButtons" name="notetypeButtons" />
        </EditorToolbar>

        {#if hint}
            <Absolute bottom right --margin="10px">
                <Notification>
                    <Badge --badge-color="tomato" --icon-align="top"
                        >{@html alertIcon}</Badge
                    >
                    <span>{@html hint}</span>
                </Notification>
            </Absolute>
        {/if}

        <Fields>
            <DecoratedElements>
                {#each fieldsData as field, index}
                    {@const content = fieldStores[index]}

                    <EditorField
                        {field}
                        {content}
                        flipInputs={plainTextDefaults[index]}
                        api={fields[index]}
                        on:focusin={() => {
                            $focusedField = fields[index];
                            bridgeCommand(`focus:${index}`);
                        }}
                        on:focusout={() => {
                            $focusedField = null;
                            bridgeCommand(
                                `blur:${index}:${getNoteId()}:${get(content)}`,
                            );
                        }}
                        on:mouseenter={() => {
                            $hoveredField = fields[index];
                        }}
                        on:mouseleave={() => {
                            $hoveredField = null;
                        }}
                        collapsed={fieldsCollapsed[index]}
                        --label-color={cols[index] === "dupe"
                            ? "var(--flag1-bg)"
                            : "var(--window-bg)"}
                    >
                        <svelte:fragment slot="field-label">
                            <LabelContainer
                                collapsed={fieldsCollapsed[index]}
                                on:toggle={async () => {
                                    fieldsCollapsed[index] = !fieldsCollapsed[index];

                                    const defaultInput = !plainTextDefaults[index]
                                        ? richTextInputs[index]
                                        : plainTextInputs[index];

                                    if (!fieldsCollapsed[index]) {
                                        refocusInput(defaultInput.api);
                                    } else if (!plainTextDefaults[index]) {
                                        plainTextsHidden[index] = true;
                                    } else {
                                        richTextsHidden[index] = true;
                                    }
                                }}
                            >
                                <svelte:fragment slot="field-name">
                                    <LabelName>
                                        {field.name}
                                    </LabelName>
                                </svelte:fragment>
                                <FieldState>
                                    {#if cols[index] === "dupe"}
                                        <DuplicateLink />
                                    {/if}
                                    {#if plainTextDefaults[index]}
                                        <RichTextBadge
                                            visible={!fieldsCollapsed[index] &&
                                                (fields[index] === $hoveredField ||
                                                    fields[index] === $focusedField)}
                                            bind:off={richTextsHidden[index]}
                                            on:toggle={async () => {
                                                richTextsHidden[index] =
                                                    !richTextsHidden[index];

                                                if (!richTextsHidden[index]) {
                                                    refocusInput(
                                                        richTextInputs[index].api,
                                                    );
                                                }
                                            }}
                                        />
                                    {:else}
                                        <PlainTextBadge
                                            visible={!fieldsCollapsed[index] &&
                                                (fields[index] === $hoveredField ||
                                                    fields[index] === $focusedField)}
                                            bind:off={plainTextsHidden[index]}
                                            on:toggle={async () => {
                                                plainTextsHidden[index] =
                                                    !plainTextsHidden[index];

                                                if (!plainTextsHidden[index]) {
                                                    refocusInput(
                                                        plainTextInputs[index].api,
                                                    );
                                                }
                                            }}
                                        />
                                    {/if}
                                    <slot
                                        name="field-state"
                                        {field}
                                        {index}
                                        visible={fields[index] === $hoveredField ||
                                            fields[index] === $focusedField}
                                    />
                                </FieldState>
                            </LabelContainer>
                        </svelte:fragment>
                        <svelte:fragment slot="rich-text-input">
                            <Collapsible
                                collapse={richTextsHidden[index]}
                                let:collapsed={hidden}
                            >
                                <RichTextInput
                                    {hidden}
                                    on:focusout={() => {
                                        saveFieldNow();
                                        $focusedInput = null;
                                    }}
                                    bind:this={richTextInputs[index]}
                                >
                                    <ImageHandle maxWidth={250} maxHeight={125} />
                                    <MathjaxHandle />
                                    <FieldDescription>
                                        {field.description}
                                    </FieldDescription>
                                </RichTextInput>
                            </Collapsible>
                        </svelte:fragment>
                        <svelte:fragment slot="plain-text-input">
                            <Collapsible
                                collapse={plainTextsHidden[index]}
                                let:collapsed={hidden}
                            >
                                <PlainTextInput
                                    {hidden}
                                    isDefault={plainTextDefaults[index]}
                                    richTextHidden={richTextsHidden[index]}
                                    on:focusout={() => {
                                        saveFieldNow();
                                        $focusedInput = null;
                                    }}
                                    bind:this={plainTextInputs[index]}
                                />
                            </Collapsible>
                        </svelte:fragment>
                    </EditorField>
                {/each}

                <MathjaxElement />
                <FrameElement />
            </DecoratedElements>
        </Fields>
    </FieldsEditor>

    <StickyContainer --gutter-block="0.1rem" --sticky-borders="1px 0 0" class="d-flex">
        <TagEditor {tags} on:tagsupdate={saveTags} />
    </StickyContainer>
</div>

<style lang="scss">
    .note-editor {
        height: 100%;
        display: flex;
        flex-direction: column;
    }
</style>
