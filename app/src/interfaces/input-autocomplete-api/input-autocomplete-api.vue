<template>
	<v-notice v-if="!url" type="warning">
		{{ t('one_or_more_options_are_missing') }}
	</v-notice>
	<div v-else>
		<v-menu attached :disabled="disabled">
			<template #activator="{ activate, deactivate }">
				<v-input
					:placeholder="placeholder"
					:disabled="disabled"
					:class="font"
					:model-value="value"
					@update:model-value="onInput"
					@focus="activate"
					@blur="deactivate"
				>
					<template v-if="iconLeft" #prepend><v-icon :name="iconLeft" /></template>
					<template v-if="iconRight" #append><v-icon :name="iconRight" /></template>
				</v-input>
			</template>

			<v-list v-if="results.length > 0">
				<v-list-item v-for="result of results" :key="result" @click="() => emitValue(result)">
					<v-list-item-content>{{ result }}</v-list-item-content>
				</v-list-item>
			</v-list>
		</v-menu>
	</div>
</template>

<script lang="ts">
import { useI18n } from 'vue-i18n';
import { defineComponent, ref, PropType } from 'vue';
import axios from 'axios';
import { throttle, get, debounce } from 'lodash';
import { render } from 'micromustache';

export default defineComponent({
	props: {
		value: {
			type: [String, Number],
			default: null,
		},
		url: {
			type: String,
			default: null,
		},
		resultsPath: {
			type: String,
			default: null,
		},
		valuePath: {
			type: String,
			default: null,
		},
		trigger: {
			type: String as PropType<'debounce' | 'throttle'>,
			default: 'throttle',
		},
		rate: {
			type: [Number, String],
			default: 500,
		},
		placeholder: {
			type: String,
			default: null,
		},
		iconLeft: {
			type: String,
			default: null,
		},
		iconRight: {
			type: String,
			default: null,
		},
		font: {
			type: String as PropType<'sans-serif' | 'serif' | 'monospace'>,
			default: 'sans-serif',
		},
		disabled: {
			type: Boolean,
			default: false,
		},
	},
	emits: ['input'],
	setup(props, { emit }) {
		const { t } = useI18n();

		const results = ref<string[]>([]);

		const fetchResultsRaw = async (value: string | null) => {
			if (!value) {
				results.value = [];
				return;
			}

			const url = render(props.url, { value });

			try {
				const result = await axios.get(url);
				const resultsArray = props.resultsPath ? get(result.data, props.resultsPath) : result.data;

				if (Array.isArray(resultsArray) === false) {
					// eslint-disable-next-line no-console
					console.warn(`Expected results type of array, "${typeof resultsArray}" received`);
					return;
				} else {
					results.value = resultsArray
						.map((result: Record<string, unknown>) => (props.valuePath ? get(result, props.valuePath) : result))
						.filter((val: unknown) => val);
				}
			} catch (err) {
				// eslint-disable-next-line no-console
				console.warn(err);
			}
		};

		const fetchResults =
			props.trigger === 'debounce'
				? debounce(fetchResultsRaw, Number(props.rate))
				: throttle(fetchResultsRaw, Number(props.rate));

		return { t, results, onInput, emitValue };

		function onInput(value: string) {
			emitValue(value);
			fetchResults(value);
		}

		function emitValue(value: string) {
			emit('input', value);
		}
	},
});
</script>

<style lang="scss" scoped>
.v-input {
	&.monospace {
		--v-input-font-family: var(--family-monospace);
	}

	&.serif {
		--v-input-font-family: var(--family-serif);
	}

	&.sans-serif {
		--v-input-font-family: var(--family-sans-serif);
	}
}
</style>
