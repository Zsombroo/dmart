<script lang="ts">
  import Attachments from "../Attachments.svelte";
  import { onDestroy, onMount } from "svelte";
  import {
    QueryType,
    RequestType,
    ResourceType,
    ResponseEntry,
    Status,
    query,
    request,
    retrieve_entry,
    ContentType,
    upload_with_payload,
    csv,
  } from "@/dmart";
  import {
      Form,
      FormGroup,
      Button,
      Modal,
      ModalBody,
      ModalFooter,
      ModalHeader,
      Label,
      Input,
      Nav,
      ButtonGroup,
      Row, TabContent, TabPane,
  } from "sveltestrap";
  import Icon from "../../Icon.svelte";
  import { _ } from "@/i18n";
  import ListView from "../ListView.svelte";
  import Prism from "../../Prism.svelte";
  import {
    JSONEditor,
    Mode,
    Validator,
    createAjvValidator,
  } from "svelte-jsoneditor";
  import { status_line } from "@/stores/management/status_line";
  import { authToken } from "@/stores/management/auth";
  import { timeAgo } from "@/utils/timeago";
  import { showToast, Level } from "@/utils/toast";
  import { faSave } from "@fortawesome/free-regular-svg-icons";
  import history_cols from "@/stores/management/list_cols_history.json";
  import refresh_spaces from "@/stores/management/refresh_spaces";
  import { website } from "@/config";
  import { isDeepEqual, removeEmpty } from "@/utils/compare";
  import metaContentSchema from "@/validations/meta.content.json";
  import SchemaEditor from "../editors/SchemaEditor.svelte";
  import checkAccess from "@/utils/checkAccess";
  import { fade } from "svelte/transition";
  import BreadCrumbLite from "../BreadCrumbLite.svelte";
  import { generateUUID } from "@/utils/uuid";
  import downloadFile from "@/utils/downloadFile";
  import { encode } from "plantuml-encoder";
  import { goto } from "@roxi/routify";
  import SchemaForm from "svelte-jsonschema-form";
  import Table2Cols from "@/components/management/Table2Cols.svelte";
  import {cleanUpSchema} from "@/utils/renderer/rendererUtils";
  import {transformToProperBodyRequest} from "@/utils/editors/schemaEditorUtils";
  import {jsonTOplantUML} from "@/utils/renderer/workflowRendererUtils";

  let header_height: number;

  export let entry: ResponseEntry;

  export let space_name: string;
  export let subpath: string;
  export let resource_type: ResourceType;
  export let schema_name: string | undefined = null;
  export let refresh = {};

  const canUpdate = checkAccess("update", space_name, subpath, resource_type);
  const canDelete = checkAccess("delete", space_name, subpath, resource_type);

  let tab_option = resource_type === ResourceType.folder ? "list" : "view";
  let content = { json: entry, text: undefined };

  let contentMeta = { json: {}, text: undefined };
  let validatorMeta: Validator = createAjvValidator({
    schema: metaContentSchema,
  });
  let oldContentMeta = { json: {}, text: undefined };

  let contentContent: any = null;
  let validator: Validator = createAjvValidator({ schema: {} });
  let entryContent: any;

  let ws = null;

  function isOpen(ws: any) {
    return ws != null && ws.readyState === ws.OPEN;
  }

  let isNeedRefresh = false;
  onMount(async () => {
    const cpy = JSON.parse(JSON.stringify(entry));
    if (entry?.payload?.content_type === "json") {
      if (contentContent === null) {
        contentContent = { json: {}, text: undefined };
      }
      contentContent.json = cpy?.payload?.body ?? {};
      contentContent = structuredClone(contentContent);
    } else {
      contentContent = cpy?.payload?.body;
    }
    delete cpy?.payload?.body;
    delete cpy?.attachments;
    contentMeta.json = cpy;
    contentMeta = structuredClone(contentMeta);
    oldContentMeta = structuredClone(contentMeta);

    if (!!entry?.payload?.body?.stream) {
      if ("websocket" in website) {
          ws = new WebSocket(`${website.websocket}?token=${$authToken}`);
      }

      ws.onopen = () => {
        ws.send(
          JSON.stringify({
            type: "notification_subscription",
            space_name: space_name,
            subpath: subpath,
          })
        );
      };

      ws.onmessage = (event) => {
        const data = JSON.parse(event?.data ?? "");
        if (data?.message?.title) {
          isNeedRefresh = true;
        }
      };
    }
  });

  onDestroy(() => {
    status_line.set("");

    if (isOpen(ws)) {
      ws.send(JSON.stringify({ type: "notification_unsubscribe" }));
    }
    if (ws != null) ws.close();
  });
  status_line.set(
    `<small>Last updated: <strong>${timeAgo(
      new Date(entry.updated_at)
    )}</strong><br/>Attachments: <strong>${
      Object.keys(entry.attachments).length
    }</strong></small>`
  );

  let schemaFormRef;
  let errorContent = null;
  async function handleSave(e: Event) {
    e.preventDefault();
    // if (!isSchemaValidated) {
    //   alert("The content does is not validated agains the schema");
    //   return;
    // }

    if (!schemaFormRef.reportValidity()) {
      return;
    }

    errorContent = null;

    const x = contentMeta.json
      ? structuredClone(contentMeta.json)
      : JSON.parse(contentMeta.text);

    let data: any = structuredClone(x);
    if (entry?.payload) {
      if (entry?.payload?.content_type === "json") {
        const y = contentContent.json
          ? structuredClone(contentContent.json)
          : JSON.parse(contentContent.text);
        if (data.payload) {
          data.payload.body = y;
        }
      } else {
        data.payload.body = contentContent;
      }
    }

    if (resource_type === ResourceType.folder) {
      const arr = subpath.split("/");
      arr[arr.length - 1] = "";
      subpath = arr.join("/");
    }
    subpath = subpath == "__root__" || subpath == "" ? "/" : subpath;

    const request_data = {
      space_name: space_name,
      request_type: RequestType.replace,
      records: [
        {
          resource_type,
          shortname: entry.shortname,
          subpath,
          attributes: data,
        },
      ],
    };

    const response = await request(request_data);
    if (response.status == Status.success) {
      showToast(Level.info);
      oldContentMeta = structuredClone(contentMeta);

      if (data.shortname !== entry.shortname) {
        const moveAttrb = {
          src_subpath: subpath,
          src_shortname: entry.shortname,
          dest_subpath: subpath,
          dest_shortname: data.shortname,
        };
        const response = await request({
          space_name: space_name,
          request_type: RequestType.move,
          records: [
            {
              resource_type,
              shortname: entry.shortname,
              subpath,
              attributes: moveAttrb,
            },
          ],
        });
        if (response.status == Status.success) {
          showToast(Level.info);
        } else {
          errorContent = response;
          showToast(Level.warn);
        }
      }

      window.location.reload();
    }
    else {
      errorContent = response;
      showToast(Level.warn);
    }
  }

  function handleRenderMenu(items: any, _context: any) {
    const separator = {
      separator: true,
    };

    const saveButton = {
      onClick: handleSave,
      icon: faSave,
      title: "Save",
    };

    const itemsWithoutSpace = items.slice(0, items.length - 2);
    return itemsWithoutSpace.concat([
      separator,
      saveButton,
      {
        space: true,
      },
    ]);
  }

  let schema = null;
  async function get_schema() {
    if (entry.payload && entry.payload.schema_shortname) {
      try {
        const schema_data: ResponseEntry = await retrieve_entry(
          ResourceType.schema,
          space_name,
          "/schema",
          entry.payload.schema_shortname,
          true,
          false
        );

        if (schema_data?.payload?.body) {
          schema = schema_data.payload.body;
          cleanUpSchema(schema.properties);
          validator = createAjvValidator({ schema });
        } else {
          schema = {};
        }
      } catch (x) {
        showToast(Level.warn, "Schema loading failed");
      }
    } else {
      schema = {};
    }
  }

  let isModalOpen = false;
  let modalFlag = "create";
  let entryType = "folder";
  let contentShortname = "";
  let workflowShortname = "";
  let selectedSchema = "";
  let selectedContentType = ContentType.json;
  let new_resource_type: ResourceType = ResourceType.content;
  let payloadFiles: FileList;

  let itemsSchemaContent: any = [
    {
      id: generateUUID(),
      name: "root",
      type: "object",
      title: "title",
      description: "",
    },
  ];

  async function handleSubmit(event: Event) {
    event.preventDefault();

    let response: any;
    if (new_resource_type === "schema") {
      let body = content.json
        ? structuredClone(content.json)
        : JSON.parse(content.text);
      body = transformToProperBodyRequest(body);
      body = body[0];
      delete body.name;

      const request_body = {
        space_name,
        request_type: RequestType.create,
        records: [
          {
            resource_type: ResourceType.schema,
            shortname: contentShortname === "" ? "auto" : contentShortname,
            subpath,
            attributes: {
              is_active: true,
              payload: {
                content_type: "json",
                schema_shortname: "meta_schema",
                body,
              },
            },
          },
        ],
      };
      response = await request(request_body);
    } else if (entryType === "content") {
      if (
        [null, "json", "text", "html", "markdown"].includes(selectedContentType)
      ) {
        let body: any;
        if (selectedContentType === "json") {
          body = entryContent.json
            ? structuredClone(entryContent.json)
            : JSON.parse(entryContent.text);
        } else {
          body = entryContent;
        }

        const request_body: any = {
          space_name,
          request_type: RequestType.create,
          records: [
            {
              resource_type: new_resource_type,
              shortname: contentShortname === "" ? "auto" : contentShortname,
              subpath,
              attributes: {
                workflow_shortname: workflowShortname,
                is_active: true,
              },
            },
          ],
        };
        if (new_resource_type === "ticket") {
          request_body.records[0].attributes.workflow_shortname =
            workflowShortname;
        }
        if (selectedContentType !== null) {
          request_body.records[0].attributes.payload = {
            content_type: selectedContentType ? selectedContentType : "json",
            schema_shortname: selectedSchema ? selectedSchema : "",
            body,
          };
        }
        response = await request(request_body);
      } else if (
        ["image", "python", "pdf", "audio", "video"].includes(
          selectedContentType
        )
      ) {
        response = await upload_with_payload(
          space_name,
          subpath,
          ResourceType[new_resource_type],
          null,
          contentShortname === "" ? "auto" : contentShortname,
          payloadFiles[0]
        );
      }
    } else if (entryType === "folder") {
      const request_body = {
        space_name,
        request_type: RequestType.create,
        records: [
          {
            resource_type: ResourceType.folder,
            shortname: contentShortname === "" ? "auto" : contentShortname,
            subpath,
            attributes: {
              is_active: true,
            },
          },
        ],
      };
      response = await request(request_body);
    }
    if (response.status === "success") {
      showToast(Level.info);
      contentShortname = "";
      isModalOpen = false;
      refresh = !refresh;
    } else {
      showToast(Level.warn);
    }
  }

  $: {
    if (schema === null && entry?.payload?.schema_shortname) {
      get_schema();
    }
  }

  async function handleDelete() {
    if (
      confirm(`Are you sure want to delete ${entry.shortname} entry`) === false
    ) {
      return;
    }

    let targetSubpath: string;
    if (resource_type === ResourceType.folder) {
      const arr = subpath.split("/");
      arr[arr.length - 1] = "";
      targetSubpath = arr.join("/");
    } else {
      targetSubpath = subpath;
    }

    const request_body = {
      space_name,
      request_type: RequestType.delete,
      records: [
        {
          resource_type,
          shortname: entry.shortname,
          subpath: targetSubpath || "/",
          branch_name: "master",
          attributes: {},
        },
      ],
    };
    const response = await request(request_body);
    if (response.status === "success") {
      showToast(Level.info);
      // await spaces.refresh();
      refresh_spaces.refresh();
      history.go(-1);
    } else {
      showToast(Level.warn);
    }
  }

  function beforeUnload(event) {
    if (!isDeepEqual(removeEmpty(contentMeta), removeEmpty(oldContentMeta))) {
      event.preventDefault();
      if (
        confirm("You have unsaved changes, do you want to leave ?") === false
      ) {
        return false;
      }
    }
  }


  let currentDiagram = {
    name: "",
    content: jsonTOplantUML(entry?.payload?.body),
    encodedContent: function () {
      return encode(this.content);
    },
  };

  $: {
    if (selectedContentType === "json") {
      entryContent = { json: {} || {}, text: undefined };
    } else {
      entryContent = "";
    }
  }

  async function handleDownload() {
    const body = {
      space_name,
      subpath,
      type: "search",
      search: "",
      retrieve_json_payload: true,
      limit: 1000,
      branch_name: "master",
      filter_schema_names: [],
    };
    const data = await csv(body);
    downloadFile(data, `${space_name}/${subpath}.csv`, "text/csv");
  }

  const toggleModal = () => {
    isModalOpen = !isModalOpen;
    contentShortname = "";
  };
</script>

<svelte:window on:beforeunload={beforeUnload} />

<Modal
  isOpen={isModalOpen}
  toggle={toggleModal}
  size={new_resource_type === "schema" ? "xl" : "lg"}
>
  <ModalHeader toggle={toggleModal} />
  <Form on:submit={async (e) => await handleSubmit(e)}>
    <ModalBody>
      <FormGroup>
        <h4>
          Creating a entry under <span class="text-success">{space_name}</span
          >/<span class="text-primary">{subpath}</span>
        </h4>
        {#if modalFlag === "create"}
          <Label class="mt-3">Resource type</Label>
          <Input bind:value={new_resource_type} type="select">
            {#each Object.values(ResourceType) as type}
              <option value={type}>{type}</option>
            {/each}
          </Input>

          {#if new_resource_type !== "schema"}
            <Label class="mt-3">Content type</Label>
            <Input bind:value={selectedContentType} type="select">
              <option value={null}>{"None"}</option>
              {#each Object.values(ContentType) as type}
                <option value={type}>{type}</option>
              {/each}
            </Input>
            <Label class="mt-3">Schema</Label>
            <Input bind:value={selectedSchema} type="select">
              <option value={null}>{"None"}</option>
              {#await query( { space_name, type: QueryType.search, subpath: "/schema", search: "", retrieve_json_payload: true, limit: 99 } ) then schemas}
                {#each schemas.records.map((e) => e.shortname) as schema}
                  <option value={schema}>{schema}</option>
                {/each}
              {/await}
            </Input>
            {#if new_resource_type === "ticket"}
              <Label class="mt-3">Workflow Shortname</Label>
              <Input
                placeholder="Shortname..."
                bind:value={workflowShortname}
              />
            {/if}
          {/if}
        {/if}
        {#if entryType === "content" && modalFlag === "create"}
          <Label class="mt-3">Shortname</Label>
          <Input placeholder="Shortname..." bind:value={contentShortname} />
          <hr />

          {#if new_resource_type === "schema"}
            <SchemaEditor bind:content />
            <Row>
              {#if errorContent}
                <h3 class="mt-3">Error:</h3>
                <Prism bind:code={errorContent} />
              {/if}
            </Row>
          {:else if selectedContentType}
            <Label class="mt-3">Content</Label>
            {#if ["image", "python", "pdf", "audio", "video"].includes(selectedContentType)}
              <Input
                accept="image/png, image/jpeg"
                bind:files={payloadFiles}
                type="file"
              />
            {/if}
            {#if selectedContentType === "json"}
              <JSONEditor mode={Mode.text} bind:content={entryContent} />
            {/if}
          {/if}
          <hr />

          <!-- <Label>Schema</Label>
            <ContentJsonEditor
              bind:self={refJsonEditor}
              content={contentSchema}
              readOnly={true}
              mode={Mode.tree}
            /> -->
        {/if}
        {#if entryType === "folder"}
          <Label class="mt-3">Shortname</Label>
          <Input
            placeholder="Shortname..."
            bind:value={contentShortname}
            required
          />
          {#if modalFlag === "update"}
            <Label class="mt-3">Content</Label>
          {/if}
        {/if}
      </FormGroup>
    </ModalBody>
    <ModalFooter>
      <Button
        type="button"
        color="secondary"
        on:click={() => {
          isModalOpen = false;
          contentShortname = "";
        }}>cancel</Button
      >
      <Button type="submit" color="primary">Submit</Button>
    </ModalFooter>
  </Form>
</Modal>

<div
  bind:clientHeight={header_height}
  class="pt-3 pb-2 px-2"
  transition:fade={{ delay: 25 }}
>
  <Nav class="w-100">
    <BreadCrumbLite
      {space_name}
      {subpath}
      {resource_type}
      {schema_name}
      shortname={entry.shortname}
    />
    <ButtonGroup size="sm" class="ms-auto align-items-center">
      <span class="ps-2 pe-1"> {$_("views")} </span>
      {#if resource_type === ResourceType.folder}
        <Button
          outline
          color="success"
          size="sm"
          class="justify-content-center text-center py-0 px-1"
          active={"list" === tab_option}
          title={$_("list")}
          on:click={() => (tab_option = "list")}
        >
          <Icon name="card-list" />
        </Button>
      {/if}

      <Button
        outline
        color="success"
        size="sm"
        class="justify-content-center text-center py-0 px-1"
        active={"view" === tab_option}
        title={$_("view")}
        on:click={() => (tab_option = "view")}
      >
        <Icon name="binoculars" />
      </Button>

      {#if canUpdate}
        <Button
          outline
          color="success"
          size="sm"
          class="justify-content-center text-center py-0 px-1"
          active={"edit_meta" === tab_option}
          title={$_("edit") + " meta"}
          on:click={() => (tab_option = "edit_meta")}
        >
          <Icon name="code-slash" />
        </Button>
        {#if entry.payload}
          <Button
            outline
            color="success"
            size="sm"
            class="justify-content-center text-center py-0 px-1"
            active={"edit_content" === tab_option}
            title={$_("edit") + " payload"}
            on:click={() => (tab_option = "edit_content")}
          >
            <Icon name="pencil" />
          </Button>
          {#if schema}
            <Button
              outline
              color="success"
              size="sm"
              class="justify-content-center text-center py-0 px-1"
              active={"edit_content_form" === tab_option}
              title={$_("edit") + " payload"}
              on:click={() => (tab_option = "edit_content_form")}
            >
              <Icon name="pencil-square" />
            </Button>
          {/if}
        {/if}
        {#if entry.payload}
          <Button
            outline
            color="success"
            size="sm"
            class="justify-content-center text-center py-0 px-1"
            active={"workflow" === tab_option}
            title={$_("edit") + " payload"}
            on:click={() => (tab_option = "workflow")}
          >
            <Icon name="diagram-3" />
          </Button>
        {/if}
      {/if}

      <Button
        outline
        color="success"
        size="sm"
        class="justify-content-center text-center py-0 px-1"
        active={"attachments" === tab_option}
        title={$_("attachments")}
        on:click={() => (tab_option = "attachments")}
      >
        <Icon name="paperclip" />
      </Button>
      <Button
        outline
        color="success"
        size="sm"
        class="justify-content-center text-center py-0 px-1"
        active={"history" === tab_option}
        title={$_("history")}
        on:click={() => (tab_option = "history")}
      >
        <Icon name="clock-history" />
      </Button>
    </ButtonGroup>
    <ButtonGroup size="sm" class="align-items-center">
      <span class="ps-2 pe-1"> {$_("actions")} </span>
      {#if canDelete}
        <Button
          outline
          color="success"
          size="sm"
          title={$_("delete")}
          on:click={handleDelete}
          class="justify-content-center text-center py-0 px-1"
        >
          <Icon name="trash" />
        </Button>
      {/if}
      <Button
        outline
        color="success"
        size="sm"
        title={$_("download")}
        on:click={handleDownload}
        class="justify-content-center text-center py-0 px-1"
      >
        <Icon name="cloud-download" />
      </Button>
    </ButtonGroup>
    {#if resource_type === ResourceType.folder}
      <ButtonGroup>
        <Button
          outline
          color="success"
          size="sm"
          title={$_("create")}
          class="justify-contnet-center text-center py-0 px-1"
          on:click={() => {
            isModalOpen = true;
            entryType = "content";
          }}
          ><Icon name="file-plus" />
        </Button>
        <Button
          outline
          color="success"
          size="sm"
          title={$_("create")}
          class="justify-contnet-center text-center py-0 px-1"
          on:click={() => {
            isModalOpen = true;
            entryType = "folder";
          }}
          ><Icon name="folder-plus" />
        </Button>
        <Button
          outline={!isNeedRefresh}
          color={isNeedRefresh ? "danger" : "success"}
          size="sm"
          title={$_("refresh")}
          class="justify-contnet-center text-center py-0 px-1"
          on:click={() => {
            refresh = !refresh;
          }}
          ><Icon name="arrow-clockwise" />
        </Button>
      </ButtonGroup>
    {/if}
  </Nav>
</div>
<div
  class="px-1 tab-content"
  style="height: calc(100% - {header_height}px); overflow: hidden auto;"
  transition:fade={{ delay: 25 }}
>
  <div class="tab-pane" class:active={tab_option === "list"}>
    <ListView {space_name} {subpath}
              folderColumns={entry?.payload?.body?.index_attributes ?? null}
              sort_by={entry?.payload?.body?.sort_by ?? null}
              sort_order={entry?.payload?.body?.sort_order ?? null}
    />
  </div>
  <div class="tab-pane" class:active={tab_option === "source"}>
    <!--JSONEditor json={entry} /-->
    <div
      class="px-1 pb-1 h-100"
      style="text-align: left; direction: ltr; overflow: hidden auto;"
    >
      <pre>
        {JSON.stringify(entry, undefined, 1)}
      </pre>
    </div>
  </div>
  <div class="tab-pane" class:active={tab_option === "view"}>
    <div
      class="px-1 pb-1 h-100"
      style="text-align: left; direction: ltr; overflow: hidden auto;"
    >
      <TabContent>
        <TabPane tabId="table" tab="Table" active><Table2Cols {entry} /></TabPane>
        <TabPane tabId="form" tab="Raw">
          <Prism code={entry} />
        </TabPane>
      </TabContent>
    </div>
  </div>
  <div class="tab-pane" class:active={tab_option === "edit_meta"}>
    {#if tab_option === "edit_meta"}
      <div
        class="px-1 pb-1 h-100"
        style="text-align: left; direction: ltr; overflow: hidden auto;"
      >
        <JSONEditor
          mode={Mode.text}
          bind:content={contentMeta}
          onRenderMenu={handleRenderMenu}
          bind:validator={validatorMeta}
        />
        {#if errorContent}
          <h3 class="mt-3">Error:</h3>
          <Prism bind:code={errorContent} />
        {/if}
      </div>
    {/if}
  </div>
  {#if entry.payload}
    <div class="tab-pane" class:active={tab_option === "edit_content"}>
      <div
        class="px-1 pb-1 h-100"
        style="text-align: left; direction: ltr; overflow: hidden auto;"
      >
        {#if entry.payload.content_type === "json" && typeof contentContent === "object" && contentContent !== null}
          <JSONEditor
            mode={Mode.text}
            bind:content={contentContent}
            bind:validator
            onRenderMenu={handleRenderMenu}
          />
        {/if}

        {#if errorContent}
          <h3 class="mt-3">Error:</h3>
          <Prism bind:code={errorContent} />
        {/if}
      </div>
    </div>
    {#if schema}
      <div class="tab-pane" class:active={tab_option === "edit_content_form"}>
        <div class="d-flex justify-content-end my-1">
          <Button on:click={handleSave}>Save</Button>
        </div>
        <div class="px-1 pb-1 h-100">
          <SchemaForm
            bind:ref={schemaFormRef}
            {schema}
            bind:data={contentContent.json}
          />
        </div>
      </div>
    {/if}
  {/if}
  {#if entry.payload}
    <div class="tab-pane" class:active={tab_option === "workflow"}>
      <div
        class="px-1 pb-1 h-100"
        style="text-align: left; direction: ltr; overflow: hidden auto;"
      >
        <div class="preview">
          <a
            href={"https://www.plantuml.com/plantuml/svg/" +
              currentDiagram.encodedContent()}
            download="{entry.shortname}.svg"
          >
            <img
              src={"https://www.plantuml.com/plantuml/svg/" +
                currentDiagram.encodedContent()}
              alt={entry.shortname}
            />
          </a>
        </div>
      </div>
    </div>
  {/if}

  <div class="tab-pane" class:active={tab_option === "history"}>
    {#key tab_option}
      <ListView
        {space_name}
        {subpath}
        type={QueryType.history}
        shortname={entry.shortname}
        is_clickable={false}
        columns={history_cols}
      />
    {/key}
    <!--History subpath="{entry.subpath}" shortname="{entry.shortname}" /-->
  </div>
  <div class="tab-pane" class:active={tab_option === "attachments"}>
    <Attachments
      {space_name}
      {subpath}
      parent_shortname={entry.shortname}
      attachments={Object.values(entry.attachments)}
    />
  </div>
</div>

<style>
  span {
    color: dimgrey;
  }
  .preview {
    width: 100%;
    text-align: center;
  }
</style>
