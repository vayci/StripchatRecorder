<!--
    主播卡片组件 / Streamer Card Component

    展示单个主播的缩略图、在线状态、观看人数和录制控制按钮。
    通过 useFastThumbnail 对多个 CDN 域名进行竞速，加快缩略图加载速度。

    Displays a single streamer's thumbnail, online status, viewer count, and recording controls.
    Uses useFastThumbnail to race multiple CDN domains for faster thumbnail loading.

    Props:
        streamer - 主播数据对象 / Streamer data object

    Emits:
        remove       - 用户点击移除按钮 / User clicks remove button
        start        - 用户点击开始录制 / User clicks start recording
        stop         - 用户点击停止录制 / User clicks stop recording
        toggle-auto  - 用户切换自动录制开关 / User toggles auto-record switch
-->
<script setup lang="ts">
	import type { StreamerEntry } from "../stores/streamers";
	import { Card, CardContent } from "@/components/ui/card";
	import { Badge } from "@/components/ui/badge";
	import { Button } from "@/components/ui/button";
	import { Switch } from "@/components/ui/switch";
	import { Label } from "@/components/ui/label";
	import { ref, watch, computed } from "vue";
	import { useFastThumbnail } from "@/composables/useFastThumbnail";
	import { copyToClipboard } from "@/lib/utils";
	import { X, Circle, Eye, Copy, Check } from "@lucide/vue";
	import { useI18n } from "vue-i18n";

	const props = defineProps<{ streamer: StreamerEntry }>();
	void props;
	const emit = defineEmits<{
		remove: [];
		start: [];
		stop: [];
		"toggle-auto": [enabled: boolean];
	}>();
	const { t } = useI18n();

	const autoRecord = ref(props.streamer.auto_record);
	watch(
		() => props.streamer.auto_record,
		(val) => { autoRecord.value = val; },
	);

	const thumbnailSrc = computed(() => props.streamer.thumbnail_url ?? null);
	const fastThumbnail = useFastThumbnail(thumbnailSrc);
	// 主播原始页面地址
	const profileUrl = computed(
		() => `https://zh.stripchat.com/${encodeURIComponent(props.streamer.username)}`,
	);

	const copied = ref(false);

	// 转发流地址 / Stream relay URL
	const streamUrl = computed(
		() => `${window.location.origin}/stream/${props.streamer.username}`,
	);

	async function copyStreamUrl() {
		try {
			await copyToClipboard(streamUrl.value);
			copied.value = true;
			setTimeout(() => { copied.value = false; }, 2000);
		} catch {}
	}

	function onAutoChange(val: boolean) {
		autoRecord.value = val;
		emit("toggle-auto", val);
	}

	function statusClass(s: StreamerEntry): string {
		if (!s.is_online) return "bg-zinc-800 text-zinc-400 border-transparent";
		if (s.status === "公开秀") return "bg-green-900 text-green-300 border-transparent";
		return "bg-amber-900 text-amber-300 border-transparent";
	}
</script>

<template>
	<Card
		class="overflow-hidden transition-colors py-0"
		:class="{
			'border-green-900/50': streamer.is_online && !streamer.is_recording,
			'border-red-900/50': streamer.is_recording,
		}"
	>
		<div class="relative aspect-video bg-muted overflow-hidden">
			<img
				v-if="fastThumbnail"
				:src="fastThumbnail"
				loading="lazy"
				class="w-full h-full object-cover"
			/>
			<div
				v-else
				class="w-full h-full flex items-center justify-center text-2xl font-bold text-muted-foreground/20 sm:text-4xl"
			>
				{{ streamer.username[0].toUpperCase() }}
			</div>
			<Circle
				v-if="streamer.is_recording"
				class="absolute top-1.5 right-2 size-2.5 fill-red-500 text-red-500 animate-pulse"
			/>
		</div>

		<CardContent class="p-2 flex flex-col gap-1.5 sm:p-3 sm:gap-2">
			<div class="flex items-center justify-between gap-1">
				<a
					:href="profileUrl"
					target="_blank"
					rel="noopener noreferrer"
					class="min-w-0 flex-1 truncate text-xs font-semibold hover:underline sm:text-sm"
					:title="streamer.username"
				>
					{{ streamer.username }}
				</a>
				<Button
					variant="ghost"
					size="icon"
					class="h-5 w-5 shrink-0 text-muted-foreground hover:text-destructive sm:h-6 sm:w-6"
					:title="t('streamerCard.removeTitle')"
					@click="emit('remove')"
				>
					<X class="size-3 sm:size-3.5" />
				</Button>
			</div>

			<div class="flex items-center gap-1 flex-wrap sm:gap-1.5">
				<Badge :class="[statusClass(streamer), 'max-w-full truncate px-1.5 text-[10px] sm:px-2 sm:text-xs']">
					{{ streamer.is_online ? streamer.status : t("streamerCard.offline") }}
				</Badge>
				<Badge v-if="streamer.is_recording" variant="destructive" class="px-1.5 text-[10px] sm:px-2 sm:text-xs">{{ t("streamerCard.recording") }}</Badge>
				<span
					v-if="streamer.is_online && streamer.viewers > 0"
					class="text-[10px] text-muted-foreground flex items-center gap-1 sm:text-xs"
				>
					<Eye class="size-3" /> {{ streamer.viewers.toLocaleString() }}
				</span>
			</div>

			<div class="flex flex-col items-stretch gap-1.5 mt-0.5 sm:flex-row sm:items-center sm:gap-2">
				<Button
					v-if="!streamer.is_recording"
					size="sm"
					class="w-full h-7 px-1 text-xs sm:h-8 sm:flex-1 sm:px-3"
					:disabled="!streamer.is_recordable"
					:title="!streamer.is_recordable ? streamer.status : ''"
					@click="emit('start')"
				>
					{{ t("streamerCard.startRecording") }}
				</Button>
				<Button
					v-else
					size="sm"
					variant="destructive"
					class="w-full h-7 px-1 text-xs sm:h-8 sm:flex-1 sm:px-3"
					@click="emit('stop')"
				>
					{{ t("streamerCard.stopRecording") }}
				</Button>

				<div class="flex items-center justify-between gap-1 shrink-0 sm:justify-start sm:gap-1.5" :title="t('streamerCard.autoRecordTitle')">
					<Switch
						:id="`auto-${streamer.username}`"
						:model-value="autoRecord"
						@update:model-value="onAutoChange"
					/>
					<Label
						:for="`auto-${streamer.username}`"
						class="text-[10px] text-muted-foreground select-none sm:text-xs"
					>
						{{ t("streamerCard.autoRecord") }}
					</Label>
				</div>
			</div>

			<!-- 转发流地址（始终显示）/ Stream URL (always shown) -->
			<div class="flex items-center gap-1 sm:gap-2">
				<div
					class="flex-1 text-[10px] font-mono text-blue-400/60 bg-blue-950/10 rounded px-1 py-1 truncate select-all sm:px-2 sm:text-xs"
					:title="t('streamerCard.streamUrlHint')"
				>
					{{ streamUrl }}
				</div>
				<Button
					size="sm"
					variant="ghost"
					class="shrink-0 px-1.5 h-6 text-muted-foreground hover:text-blue-300 sm:px-2"
					:title="t('streamerCard.copyStreamUrl')"
					@click="copyStreamUrl"
				>
					<Check v-if="copied" class="size-3 text-green-400" />
					<Copy v-else class="size-3" />
				</Button>
			</div>
		</CardContent>
	</Card>
</template>

