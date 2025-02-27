<template>
	<div class="barcode-format-selector">
				<AppPropertiesFrappeControl
					:key="`FC_${fd.name}`"
					class="barcode-format"
					:field="fd"
				/>
			</div>
	<div class="preview-container">
		<div
		class="settings"
		@click.self.stop="selectedEl = null"
		>
			<template v-for="(field, index) in fieldnames" :key="index">
				<div
					:class="[
						'dynamic-field',
						field.fieldtype == 'StaticText' && 'static-text',
						selectedEl?.field == field &&
							(field.fieldtype == 'StaticText'
								? 'static-text-selected'
								: 'dynamic-field-selected'),
					]"
					:value="field.fieldname"
					@click.stop="handleClick($event, field, index)"
				>
					<span
						v-if="!field.is_static && field.is_labelled"
						:contenteditable="contenteditable"
						:class="['label-text', selectedEl?.field == field && 'label-text-selected']"
						:ref="(el) => (field.labelRef = el)"
						@dblclick="handleDblClick($event, field)"
						@blur="handleBlur($event, field)"
						v-html="
							`${field.label}` ||
							`{{ ${field.parentField ? field.parentField + '.' : ''}${
								field.fieldname
							} }}`
						"
					></span>
					<span
						:ref="(el) => (field.spanRef = el)"
						class="preview-text"
						:class="{ 'preview-text-selected': selectedEl?.field == field }"
						:contenteditable="contenteditable"
						@dblclick="handleDblClick($event, field)"
						@blur="handleBlur($event, field)"
						v-html="
							field.value ||
							(field.is_static
								? 'Add Text'
								: `{{ ${field.parentField ? field.parentField + '.' : ''}${
										field.fieldname
								  } }}`)
						"
					></span>
					<span v-if="field.nextLine" class="next-line fa fa-level-down"></span>
				</div>
				<br v-if="field.nextLine" />
			</template>
		</div>
		<div class="preview">
			<div>
				<div
				:style="[
					style
				]"
				:class="['barcode', classes]"
				:key="id"
				v-html="barcodeSvg || ''"
			></div>
			</div>
		</div>
	</div>
	<div class="footer">
		<div class="icons" v-if="!fieldnames[0]?.is_static">
			<div @click="addStaticText">
				<em style="font-weight: 900">T</em>
				<sub style="font-weight: 600; font-size: 1em bottom:-0.15em">+</sub>
				<span style="font-size: 12px; padding: 0px 5px">Static Text</span>
			</div>
		</div>
	</div>
</template>

<script setup>
import { ref, watch, toRefs, nextTick, onMounted } from "vue";
import { useMainStore } from "../../store/MainStore";
import { makeFeild } from "../../frappeControl";
import { storeToRefs } from "pinia";
import AppPropertiesFrappeControl from "./AppPropertiesFrappeControl.vue";
const MainStore = useMainStore();
const props = defineProps({
	fieldnames: {
		type: Array,
		required: true,
	},
	modalLabel: {
		type: String,
		default: "",
	},
	isTable: {
		type: Boolean,
		default: false,
	},
});
const label = ref("");
const barcodeSvg = ref(null);
const setLabel = (value) => {
	if (label.value != value) {
		label.value = value;
	}
};
const { barcodeColor, barcodeBackgroundColor, isDynamic, barcodeFormat, barcodeShowText, style} = toRefs(MainStore.getCurrentElementsValues[0]);
onMounted(() => {
	label.value = props.modalLabel;
});

const fd = {
		label: "Format:",
		name: "barcodeFormatModal",
		isLabelled: true,
		condtional: null,
		frappeControl: (ref, name) => {
			const MainStore = useMainStore();
			const { barcodeFormats } = storeToRefs(MainStore);
			makeFeild({
				name: name,
				ref: ref,
				fieldtype: "Autocomplete",
				requiredData: [
					barcodeFormats,
					MainStore.getCurrentElementsValues[0] ||
						MainStore.globalStyles["barcode"],
				],
				options: () => barcodeFormats.value,
				reactiveObject: () => {
					return (
						MainStore.getCurrentElementsValues[0] ||
						MainStore.globalStyles["barcode"]
					);
				},
				propertyName: "barcodeFormat",
			});
		},
	}

const setBarcode = async () => {
	try {
		const options = {
			background : barcodeBackgroundColor.value || "#ffffff",
			quiet_zone: 1,
		};
		if (barcodeFormat.value == "qrcode") {
			options["module_color"] = barcodeColor.value || "#000000";
		} else {
			options["foreground"] = barcodeColor.value || "#000000";
			options["show_text"] = barcodeShowText.value || "Yes";
		}
		let barcode = await frappe.call(
			"print_designer.print_designer.page.print_designer.print_designer.get_barcode",
			{
				barcode_format: barcodeFormat.value,
				barcode_value: props.fieldnames[0].value,
				options,
				height: 80.0,
			}
		);
		barcodeSvg.value = barcode.message.value;
	} catch (e) {
		barcodeSvg.value = null;
	}
};

watch(() => [props.fieldnames, barcodeFormat.value, barcodeShowText.value], () => setBarcode(), { deep: true, immediate: true });

const parentField = ref("");
const setParentField = (value) => {
	if (parentField.value != value) {
		parentField.value = value;
	}
};
const selectedEl = ref(null);
const setSelectedEl = (value) => {
	if (selectedEl.value != value) {
		selectedEl.value = value;
	}
};
const contenteditable = ref(false);
defineExpose({ parentField, setParentField, selectedEl, setSelectedEl, label, setLabel });

const handleClick = (event, field, index) => {
	selectedEl.value = { index, field };
	parentField.value = field.parentField;
};

const selectElementContents = (el) => {
	const range = document.createRange();
	range.selectNodeContents(el);
	const sel = window.getSelection();
	sel.removeAllRanges();
	sel.addRange(range);
};

const handleDblClick = (event, field) => {
	if (field.fieldtype != "StaticText" && !field.is_labelled) return;
	contenteditable.value = true;
	setTimeout(() => {
		event.target.focus();
		selectElementContents(field.labelRef || field.spanRef);
	}, 0);
};

const handleBlur = (event, field) => {
	contenteditable.value = false;
	if (event.target == field.spanRef) {
		field.value = event.target.innerHTML;
	} else {
		field.label = event.target.innerHTML;
	}
};

const addStaticText = (event) => {
	let newText = {
		doctype: "",
		parentField: "",
		fieldname: frappe.utils.get_random(10),
		value: "Add Text",
		fieldtype: "StaticText",
		is_static: true,
		is_labelled: false,
		nextLine: false,
		style: {},
	};
	props.fieldnames.splice(0, 1, newText);
	contenteditable.value = true;
	isDynamic.value = false;
	nextTick(() => {
		props.fieldnames[0].spanRef.focus();
		selectElementContents(props.fieldnames[0].spanRef);
		selectedEl.value = { index: 0, field: newText };
	});
};
</script>

<style lang="scss" scoped>
.preview-container {
	flex: 1;
	display: flex;
	padding: 0px 15px;
	border: 1px solid var(--gray-200);
	border-radius: var(--border-radius);
	height: calc(23vh - 45px);
	min-height: 100px;
	width: 100%;
	background-color: var(--bg-color);
	overflow: auto;
	margin-top: 15px;
	padding-top: 10px;
	.settings {
		flex: 50%;
	}

	.dynamic-field {
		width: fit-content;
		display: inline-block;
		vertical-align: top;
		box-sizing: border-box;
		padding: 8px 1px;
		color: var(--text-light);

		&:hover,
		&.dynamic-field-selected {
			border: 1px solid var(--primary);
			padding: 7px 0;
			border-radius: var(--border-radius);
		}

		.label-text {
			padding: 6px 1px;
			margin: 0px 3px;
			&:hover,
			&.label-text-selected {
				border: 1px solid var(--gray-900);
				padding: 6px 1px;
				margin: 0px 2px;
				border-radius: var(--border-radius);
			}
		}

		&.static-text:hover,
		&.static-text-selected {
			border: 1px solid var(--gray-900);
			padding: 7px 0;
			border-radius: var(--border-radius);
		}

		.preview-text {
			padding: 0;
			padding-left: 2px;
			&:hover,
			&.preview-text-selected {
				color: var(--dark);
			}
		}

		.next-line {
			margin: 0px 7px;
			color: var(--text-light);
		}
	}
}
.footer {
	display: flex;
	align-items: center;
	justify-content: space-between;
	.icons {
		display: flex;
		flex: 1;
		align-items: center;
		font-size: 16px;
		justify-content: flex-start;
		padding: 10px;
		color: var(--text-light);

		& > * {
			font-weight: 500;
			padding: 0px 10px;
			border-left: 1px solid var(--gray-300);
			&:first-child {
				padding-left: 0px;
				border-left: 0px;
			}
		}
	}
	.deleteIcon {
		display: flex;
		flex: 1;
		max-width: 60px;
		margin-right: 20px;
		align-items: center;
		font-size: medium;
		justify-content: center;
		padding: 10px 20px;
		margin-right: 6px;
		color: var(--danger);
	}
}

.dropzone {
	background-color: var(--gray-200);
}
</style>
<style lang="scss" deep>
.preview {
	flex: 50%;
	.barcode {
		text-align: right;
	}
	.print-qrcode {
		max-width: 120px;
		max-height: 120px;
	}
}
</style>
