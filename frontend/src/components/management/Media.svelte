<script lang="ts">
  import { JSONEditor, Mode } from "svelte-jsoneditor";
  import { ResourceType } from "@/dmart";

  export let attributes: any = {};
  export let resource_type: ResourceType;
  export let url: string;
  export let displayname: string = undefined;
  let content_type: string = attributes?.payload?.content_type || "";
  let body: any = attributes?.payload?.body;
1
</script>

{#if resource_type === ResourceType.comment}
  <div>
    <p style="margin: 0px"><b>State:</b>{attributes.state}</p>
    <br />
    <p style="margin: 0px"><b>Body:</b>{attributes.body}</p>
  </div>
{:else if content_type.includes("json")}
  <JSONEditor mode={Mode.text} content={{ json: body, text: undefined }} readOnly={true} />
{:else if content_type.includes("image")}
  <img src={url} alt={displayname} class="mw-100 border" />
{:else if content_type.includes("audio")}
  <audio controls src={url}>
    <track kind="captions" />
  </audio>
{:else if content_type.includes("video")}
  <video controls src={url}>
    <track kind="captions" />
  </video>
{:else if content_type.includes("pdf")}
  <div
    class="h-100 w-100 vh-100 embed-responsive embed-responsive-16by9"
    style="overflow: hidden hidden;"
  >
    <object
      title={displayname}
      class="h-100 w-100 embed-responsive-item"
      type="application/pdf"
      style="height: 100vh;"
      data={url}
    >
      <p>For some reason PDF is not rendered here properly.</p>
    </object>
  </div>
{:else}
  <a href={url} title={displayname}
     target="_blank" rel="noopener noreferrer">link {displayname}</a>
{/if}
