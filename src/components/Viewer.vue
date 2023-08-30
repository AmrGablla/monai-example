<template>
  <input
    type="file"
    multiple
    id="selectDicomFile"
    style="margin-bottom: 20px"
    @change="handleFileChange"
  />
  <input
    type="file"
    multiple
    id="selectSegFile"
    style="margin-bottom: 20px"
    @change="handleSegFileChange"
  />

  <div id="content"></div>
</template>

<script>
import cornerstoneDICOMImageLoader from "@cornerstonejs/dicom-image-loader";
import cornerstone from "cornerstone-core";
import SegmentationReader from "@/monai/SegmentationReader";
import {
  createSegment,
  updateSegment,
  getLabelMaps,
  flattenLabelmaps,
} from "@/monai/SegmentationUtils";
import { getLabelColor } from "@/monai/GenericUtils";
import cornerstoneTools from "cornerstone-tools";
import dicomParser from "dicom-parser";
import Hammer from "hammerjs";
import cornerstoneMath from "cornerstone-math";
import { DeepgrowProbeTool } from "@/tools/DeepgrowProbeTool";

export default {
  name: "ViewerComponent",
  data() {
    return {
      toolGroupId: "myToolGroup",
      viewport: null,
      element: null,
      renderingEngine: {},
      viewportId: "CT_STACK",
      numberOfFrames: 0,
      labels: {
        pancreas: 1,
        "pancreatic tumor": 2,
      },
    };
  },
  async mounted() {
    cornerstoneDICOMImageLoader.external.cornerstone = cornerstone;
    cornerstoneDICOMImageLoader.external.dicomParser = dicomParser;
    cornerstoneTools.external.cornerstone = cornerstone;
    cornerstoneTools.external.Hammer = Hammer;
    cornerstoneTools.external.cornerstoneMath = cornerstoneMath;

    cornerstoneTools.init();

    const content = document.getElementById("content");

    // image div
    const element = document.createElement("div");
    element.oncontextmenu = (e) => e.preventDefault();
    element.id = "cornerstone-element";
    element.style.width = "500px";
    element.style.height = "500px";

    const div = document.createElement("div");
    div.style.display = "flex";
    div.style.flexDirection = "row";

    div.appendChild(element);

    content.appendChild(div);

    cornerstone.enable(element);

    this.element = element;

    cornerstoneTools.addTool(DeepgrowProbeTool);
    cornerstoneTools.addTool(cornerstoneTools.BrushTool, {
      name: "BrushEraser",
      configuration: {
        alwaysEraseOnClick: true,
      },
    });
    cornerstoneTools.addTool(cornerstoneTools.CircleScissorsTool, {
      name: "CircleScissorsEraser",
      defaultStrategy: "ERASE_INSIDE",
    });
    cornerstoneTools.addTool(cornerstoneTools.FreehandScissorsTool, {
      name: "FreehandScissorsEraser",
      defaultStrategy: "ERASE_INSIDE",
    });
    cornerstoneTools.addTool(cornerstoneTools.RectangleScissorsTool, {
      name: "RectangleScissorsEraser",
      defaultStrategy: "ERASE_INSIDE",
    });
    cornerstoneTools.addTool(cornerstoneTools.StackScrollMouseWheelTool);
    cornerstoneTools.setToolActive("StackScrollMouseWheel", {});
  },
  methods: {
    refreshSegTable(id) {
      const element = this.element;

      const labelmaps = getLabelMaps(element);
      const segments = flattenLabelmaps(labelmaps);
      if (!segments.length) {
        id = undefined;
      } else if (!id) {
        id = segments[segments.length - 1].id;
      }

      // do not select if index is from a scribbles volume
      // scribbles volumes exist in table
      // but wont be shown or selectable through the table ui
      for (let i = 0; i < segments.length; i++) {
        if (
          segments[i].meta.SegmentLabel.includes("scribbles") &&
          segments[i].id == id
        ) {
          id = null;
          break;
        }
      }
    },
    async onSegmentation(
      file,
      labels,
      operation,
      slice,
      overlap,
      selectedIndex = 0
    ) {
      // if (!selectedIndex) {
      //   selectedIndex = this.getSelectedActiveIndex();
      // }
      const { image } = SegmentationReader.parseNrrdData(file);

      if (labels) {
        let i = 0;
        for (var label in labels) {
          if (Array.isArray(labels)) {
            label = labels[label];
          }

          if (label === "background") {
            console.debug("Ignore Background...");
            continue;
          }
          const resp = createSegment(
            this.element,
            label,
            "",
            getLabelColor(label),
            // true
            i === 0 ? !overlap : false
          );
          if (i === 0) {
            selectedIndex = resp;
          }
          i++;
        }
      }

      if (!operation && overlap) {
        operation = "overlap";
      }

      updateSegment(
        this.element,
        selectedIndex.labelmapIndex,
        selectedIndex.segmentIndex,
        image,
        this.numberOfFrames,
        operation,
        slice
      );
    },
    async handleSegFileChange(e) {
      try {
        const file = e.target.files[0];
        const reader = new FileReader();
        reader.onloadend = async (event) => {
          await this.onSegmentation(event.target.result, this.labels);
        };
        reader.readAsArrayBuffer(file);
      } catch (err) {
        console.log(err);
      }
    },
    handleFileChange(e) {
      const imageIds = [];
      Array.from(e.target.files).forEach((file) => {
        const imageId =
          cornerstoneDICOMImageLoader.wadouri.fileManager.add(file);
        imageIds.push(imageId);
      });
      this.numberOfFrames = imageIds.length;
      cornerstone.loadImage(imageIds[0]).then((image) => {
        cornerstone.displayImage(this.element, image);
        let stack = {
          currentImageIdIndex: 0,
          imageIds: imageIds.reverse(),
        };
        cornerstoneTools.addStackStateManager(this.element, ["stack"]);
        cornerstoneTools.addToolState(this.element, "stack", stack);
      });
    },
  },
};
</script>
