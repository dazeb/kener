<script>
  import { onMount } from "svelte";
  import * as Card from "$lib/components/ui/card";
  import { base } from "$app/paths";
  import { Button } from "$lib/components/ui/button";
  import GMI from "$lib/components/gmi.svelte";
  import * as Select from "$lib/components/ui/select";
  import { Label } from "$lib/components/ui/label";
  import ExternalLink from "lucide-svelte/icons/external-link";
  import Loader from "lucide-svelte/icons/loader";

  export let data;
  let canSendEmail = data.canSendEmail;
  let page = 1;
  let limit = 25; // Set a default limit value
  let total = 0; // Initialize total to 0
  let totalPages = 0; // Initialize totalPages
  let subscriptionTrigger = {
    id: null,
    subscription_trigger_status: "INACTIVE",
    subscription_trigger_type: "email",
    config: {
      createIncident: false,
      updateIncident: false,
      insertIncidentMonitor: false,
      updateIncidentComment: false,
      insertIncidentComment: false
    }
  };
  let selectedTriggerId = null;
  let subscriberListLoading = false;
  let pageLoading = true;
  let subscribers = [];

  async function loadSubscriptionTrigger() {
    try {
      const res = await fetch(base + "/manage/app/api/", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ action: "getSubscriptionTrigger" })
      });
      let data = await res.json();
      if (data) {
        if (!!data.config) {
          data.config = JSON.parse(data.config);
        }

        subscriptionTrigger = data;
      }
    } catch (error) {
      console.log("Error fetching subscription trigger: " + error);
    }
  }

  async function saveSubscriptionTrigger() {
    try {
      const res = await fetch(base + "/manage/app/api/", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          action: "createSubscriptionTrigger",
          data: {
            subscription_trigger_type: "email",
            subscription_trigger_status: subscriptionTrigger.subscription_trigger_status,
            config: JSON.stringify(subscriptionTrigger.config)
          }
        })
      });
      //subscriptionTrigger = await res.json();
    } catch (error) {
      console.console.log(error);
    }
  }

  async function loadSubscribers() {
    subscriberListLoading = true;
    try {
      const res = await fetch(base + "/manage/app/api/", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          action: "getSubscribers",
          data: {
            page: page,
            limit: limit
          }
        })
      });
      const data = await res.json();
      subscribers = data.subscriptions || [];
      total = data.total || 0;
      totalPages = Math.ceil(total / limit); // Calculate total pages based on total and limit
    } catch (error) {
      console.error("Error fetching subscribers:", error);
      // Optionally show an error message to the user
      alert("Error loading subscribers. Please try again.");
      subscribers = [];
      total = 0;
    } finally {
      subscriberListLoading = false;
    }
  }

  onMount(async () => {
    await loadSubscriptionTrigger();
    await loadSubscribers(); // Load subscribers initially
    pageLoading = false;
  });

  function updateSubscriptionStatus(id, status) {
    // Call the API to update the subscription status
    fetch(base + "/manage/app/api/", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        action: "updateSubscriptionStatus",
        data: {
          id: id,
          status: status
        }
      })
    });
  }

  function updateSubscriptionTriggerStatus(id, status) {
    // Call the API to update the subscription status
    fetch(base + "/manage/app/api/", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        action: "updateSubscriptionTriggerStatus",
        data: {
          id: id,
          status: status
        }
      })
    });
  }

  // Reactive statement to calculate totalPages
</script>

<div class="min-h-[70vh]">
  <Card.Root class="mt-4">
    <Card.Header class="border-b">
      <Card.Title class="relative">
        Events Subscription
        {#if pageLoading}
          <Loader size="16" class="absolute right-0 top-0 animate-spin text-gray-500" />
        {:else if canSendEmail && !!subscriptionTrigger.id}
          <label class="absolute right-0 top-0 flex cursor-pointer items-center justify-between gap-2 text-sm">
            <Label class="text-xs font-medium ">
              {subscriptionTrigger.subscription_trigger_status == "ACTIVE" ? "Active" : "Inactive"}
            </Label>
            <input
              type="checkbox"
              value=""
              disabled={!canSendEmail}
              class="peer sr-only"
              checked={subscriptionTrigger.subscription_trigger_status == "ACTIVE"}
              on:change={(e) => {
                subscriptionTrigger.subscription_trigger_status = e.target.checked ? "ACTIVE" : "INACTIVE";
                // Call the API to update the subscription status
                updateSubscriptionTriggerStatus(
                  subscriptionTrigger.id,
                  subscriptionTrigger.subscription_trigger_status
                );
              }}
            />
            <div
              class="peer relative h-6 w-11 rounded-full bg-gray-200 after:absolute after:start-[2px] after:top-[2px] after:h-5 after:w-5 after:rounded-full after:border after:border-gray-300 after:bg-white after:transition-all after:content-[''] peer-checked:bg-blue-600 peer-checked:after:translate-x-full peer-checked:after:border-white peer-focus:outline-none dark:border-gray-600 dark:bg-gray-700 dark:peer-focus:ring-blue-800 rtl:peer-checked:after:-translate-x-full"
            ></div>
          </label>
        {/if}
      </Card.Title>
      <Card.Description>Configure and view who have subscribed to updates for your status page.</Card.Description>
    </Card.Header>
    <Card.Content class="py-2">
      {#if !canSendEmail}
        <div class="my-2 rounded-sm border border-destructive p-2 text-sm font-medium text-destructive">
          You have not configured your email settings yet. Please configure your email settings to send updates to your
          subscribers. You can read the documentation by visiting this link
          <a
            href="https://kener.ing/docs/environment-vars#smtp"
            target="_blank"
            rel="noopener noreferrer"
            class="text-blue-600 underline"
          >
            here
          </a>. You will either have to set up SMTP or use Resend.
        </div>
      {/if}
      <div class="mt-5 grid grid-cols-2 gap-y-2">
        <!-- Checkbox -->
        <div class="border-b pb-3">
          <label class="cursor-pointer">
            <input
              on:change={(e) => {
                subscriptionTrigger.config.createIncident = e.target.checked;
                saveSubscriptionTrigger(); // Save the subscription trigger after change
              }}
              class="mr-1"
              type="checkbox"
              checked={subscriptionTrigger.config.createIncident}
              disabled={!canSendEmail || subscriptionTrigger.subscription_trigger_status != "ACTIVE"}
            />
            Event Triggered
          </label>
          <p class="pl-6 text-xs font-medium text-muted-foreground">
            Send notification when a new event like incident or maintenance is created.
          </p>
        </div>
        <div class="border-b pb-3">
          <label class="cursor-pointer">
            <input
              on:change={(e) => {
                subscriptionTrigger.config.updateIncident = e.target.checked;
                saveSubscriptionTrigger(); // Save the subscription trigger after change
              }}
              class="mr-1"
              type="checkbox"
              checked={subscriptionTrigger.config.updateIncident}
              disabled={!canSendEmail || subscriptionTrigger.subscription_trigger_status != "ACTIVE"}
            />
            Event Updated
          </label>
          <p class="pl-6 text-xs font-medium text-muted-foreground">Send notification when an event is updated.</p>
        </div>
        <div class="border-b pb-3">
          <label class="cursor-pointer">
            <input
              on:change={(e) => {
                subscriptionTrigger.config.insertIncidentMonitor = e.target.checked;
                saveSubscriptionTrigger(); // Save the subscription trigger after change
              }}
              class="mr-1"
              type="checkbox"
              checked={subscriptionTrigger.config.insertIncidentMonitor}
              disabled={!canSendEmail || subscriptionTrigger.subscription_trigger_status != "ACTIVE"}
            />
            Update Monitor in Event
          </label>
          <p class="pl-6 text-xs font-medium text-muted-foreground">
            Send notification when a new monitor is added to an event.
          </p>
        </div>
        <div class="border-b pb-3">
          <label class="cursor-pointer">
            <input
              on:change={(e) => {
                subscriptionTrigger.config.updateIncidentComment = e.target.checked;
                saveSubscriptionTrigger(); // Save the subscription trigger after change
              }}
              class="mr-1"
              type="checkbox"
              checked={subscriptionTrigger.config.updateIncidentComment}
              disabled={!canSendEmail || subscriptionTrigger.subscription_trigger_status != "ACTIVE"}
            />
            Update Incident Comment
          </label>
          <p class="pl-6 text-xs font-medium text-muted-foreground">
            Send notification when an event comment is updated.
          </p>
        </div>
        <div class=" pb-3">
          <label class="cursor-pointer">
            <input
              on:change={(e) => {
                subscriptionTrigger.config.insertIncidentComment = e.target.checked;
                saveSubscriptionTrigger(); // Save the subscription trigger after change
              }}
              class="mr-1"
              type="checkbox"
              checked={subscriptionTrigger.config.insertIncidentComment}
              disabled={!canSendEmail || subscriptionTrigger.subscription_trigger_status != "ACTIVE"}
            />
            Insert Incident Comment
          </label>
          <p class="pl-6 text-xs font-medium text-muted-foreground">
            Send notification when a new comment is added to an event.
          </p>
        </div>
      </div>
    </Card.Content>
  </Card.Root>

  <Card.Root class="mt-4">
    <Card.Header class="border-b">
      <Card.Title class="relative">
        Site Subscribers
        <!-- Temporarily removed loader to debug -->
        <!-- {#if subscriberListLoading}
          <Loader size="16" class="absolute right-0 top-0 animate-spin text-gray-500" />
        {/if} -->
      </Card.Title>
      <Card.Description>
        View and manage subscribers who have signed up for updates on your status page.
      </Card.Description>
    </Card.Header>
    <Card.Content class="py-4">
      <div class="flex flex-col">
        <div class="-m-1.5 overflow-x-auto">
          <div class="inline-block min-w-full p-1.5 align-middle">
            <div class="overflow-hidden rounded-lg border dark:border-neutral-700">
              <table class="min-w-full divide-y divide-gray-200 dark:divide-neutral-700">
                <thead>
                  <tr>
                    <th
                      scope="col"
                      class="px-6 py-3 text-start text-xs font-medium uppercase text-gray-500 dark:text-neutral-500"
                    >
                      Email
                    </th>

                    <th
                      scope="col"
                      class="px-6 py-3 text-start text-xs font-medium uppercase text-gray-500 dark:text-neutral-500"
                    >
                      Subscribed Monitors
                    </th>
                    <th
                      scope="col"
                      class="px-6 py-3 text-center text-xs font-medium uppercase text-gray-500 dark:text-neutral-500"
                      >Subscribed At</th
                    >
                    <th scope="col" class="flex w-[130px] gap-x-2 px-1 py-3 text-start text-xs font-medium uppercase">
                      <Select.Root
                        portal={null}
                        class="w-full"
                        onSelectedChange={(e) => {
                          if (e) {
                            // Check if e is not null/undefined
                            page = e.value;
                            loadSubscribers();
                          }
                        }}
                        selected={{
                          value: page,
                          label: "Page " + page
                        }}
                      >
                        <Select.Trigger id="page" class="h-6 w-full px-2 py-1">
                          <Select.Value placeholder="Page" class="text-xs" />
                        </Select.Trigger>
                        <Select.Content class="text-xs">
                          <Select.Group>
                            <Select.Label class="text-xs">Pages</Select.Label>
                            {#if totalPages > 0}
                              {#each Array.from({ length: totalPages }, (_, i) => i + 1) as pageNum}
                                <Select.Item value={pageNum} class="text-xs font-medium uppercase"
                                  >Page {pageNum}</Select.Item
                                >
                              {/each}
                            {:else}
                              <Select.Item value={1} disabled class="text-xs font-medium uppercase">Page 1</Select.Item>
                            {/if}
                          </Select.Group>
                        </Select.Content>
                      </Select.Root>
                    </th>
                  </tr>
                </thead>
                <tbody class="divide-y divide-gray-200 dark:divide-neutral-700">
                  {#if subscriberListLoading}
                    <tr>
                      <td colspan="5" class="py-4 text-center text-sm text-gray-500">
                        <Loader size="16" class="mx-auto animate-spin" /> Loading subscribers...
                      </td>
                    </tr>
                  {:else if subscribers.length === 0}
                    <tr>
                      <td colspan="5" class="py-4 text-center text-sm text-gray-500"> No subscribers found. </td>
                    </tr>
                  {:else}
                    {#each subscribers as user}
                      <!-- Add a unique key if available, e.g., user.id -->
                      <tr>
                        <td class="whitespace-nowrap px-6 py-4 text-sm font-medium">
                          {user.subscriber.email}
                        </td>

                        <td class="whitespace-nowrap px-6 py-4 text-sm">
                          <!-- Display monitor tags comma-separated -->

                          {#if !!user.monitor?.name}
                            <div class="items-left flex gap-x-2">
                              {#if !!user.monitor.image}
                                <GMI src={user.monitor.image} alt="" classList="h-4" />
                              {/if}
                              <span class="text-sm font-medium">{user.monitor.name}</span>
                            </div>
                          {:else}
                            <span class="text-sm font-medium uppercase italic text-cyan-500">All Monitors</span>
                          {/if}
                        </td>
                        <td class="whitespace-nowrap px-6 py-4 text-center text-xs font-semibold uppercase">
                          <!-- Format date nicely if needed -->
                          {new Date(user.created_at).toLocaleString()}
                        </td>
                        <td class="whitespace-nowrap px-6 py-4 text-center text-xs font-semibold">
                          <label class="flex w-full cursor-pointer items-center justify-between">
                            <input
                              type="checkbox"
                              value=""
                              disabled={!canSendEmail || subscriptionTrigger.subscription_trigger_status != "ACTIVE"}
                              class="peer sr-only"
                              checked={user.subscriptions_status == "ACTIVE"}
                              on:change={(e) => {
                                user.subscriptions_status = e.target.checked ? "ACTIVE" : "INACTIVE";
                                // Call the API to update the subscription status
                                updateSubscriptionStatus(user.id, user.subscriptions_status);
                              }}
                            />
                            <div
                              class="peer relative h-6 w-11 rounded-full bg-gray-200 after:absolute after:start-[2px] after:top-[2px] after:h-5 after:w-5 after:rounded-full after:border after:border-gray-300 after:bg-white after:transition-all after:content-[''] peer-checked:bg-blue-600 peer-checked:after:translate-x-full peer-checked:after:border-white peer-focus:outline-none dark:border-gray-600 dark:bg-gray-700 dark:peer-focus:ring-blue-800 rtl:peer-checked:after:-translate-x-full"
                            ></div>
                          </label>
                        </td>
                      </tr>
                    {/each}
                  {/if}
                </tbody>
              </table>
            </div>
          </div>
        </div>
      </div>
    </Card.Content>
  </Card.Root>
</div>
