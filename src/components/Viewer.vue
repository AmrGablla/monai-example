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
import {
  RenderingEngine,
  setVolumesForViewports,
  Enums,
  volumeLoader,
} from "@cornerstonejs/core";
import cornerstoneDICOMImageLoader from "@cornerstonejs/dicom-image-loader";
import * as cornerstoneTools from "@cornerstonejs/tools";
// import SegmentationReader from "@/monai/SegmentationReader";
// import {
//   createSegment,
//   updateSegment,
//   getLabelMaps,
//   flattenLabelmaps,
// } from "@/monai/SegmentationUtils";
// import { getLabelColor } from "@/monai/GenericUtils";
import { prefetchMetadataInformation } from "../helpers/convertMultiframeImageIds";
import initDemo from "../helpers/initDemo";

export default {
  name: "ViewerComponent",
  data() {
    return {
      toolGroupId: "myToolGroup",
      viewport: null,
      element: null,
      renderingEngine: {},
      viewportId: "CT_ORTHOGRAPHIC",
      numberOfFrames: 0,
      toolGroup: {},
      imageIds: [],
      renderingEngineId: "myRenderingEngine",
      volumeId: "cornerstoneStreamingImageVolume:CT_VOLUME_ID",
      labels: {
        pancreas: 1,
        "pancreatic tumor": 2,
      },
    };
  },
  async beforeCreate() {
    console.log("call beforeCreate");
    await initDemo();
    this.toolGroup = cornerstoneTools.ToolGroupManager.createToolGroup(
      this.toolGroupId
    );
    cornerstoneTools.addTool(cornerstoneTools.StackScrollMouseWheelTool);
    this.toolGroup.addTool(cornerstoneTools.StackScrollMouseWheelTool.toolName);
    this.toolGroup.addTool(cornerstoneTools.BrushTool, {
      name: "BrushEraser",
      configuration: {
        alwaysEraseOnClick: true,
      },
    });
    this.toolGroup.addTool(cornerstoneTools.CircleScissorsTool, {
      name: "CircleScissorsEraser",
      defaultStrategy: "ERASE_INSIDE",
    });
    this.toolGroup.addTool(cornerstoneTools.FreehandScissorsTool, {
      name: "FreehandScissorsEraser",
      defaultStrategy: "ERASE_INSIDE",
    });
    this.toolGroup.addTool(cornerstoneTools.RectangleScissorsTool, {
      name: "RectangleScissorsEraser",
      defaultStrategy: "ERASE_INSIDE",
    });
    this.toolGroup.setToolActive(
      cornerstoneTools.StackScrollMouseWheelTool.toolName
    );
    // Instantiate a rendering engine
    this.renderingEngine = new RenderingEngine(this.renderingEngineId);
  },
  mounted() {
    console.log("mounted");
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

    this.element = element;
  },
  methods: {
    // refreshSegTable(id) {
    //   const element = this.element;

    //   const labelmaps = getLabelMaps(element);
    //   const segments = flattenLabelmaps(labelmaps);
    //   if (!segments.length) {
    //     id = undefined;
    //   } else if (!id) {
    //     id = segments[segments.length - 1].id;
    //   }

    //   // do not select if index is from a scribbles volume
    //   // scribbles volumes exist in table
    //   // but wont be shown or selectable through the table ui
    //   for (let i = 0; i < segments.length; i++) {
    //     if (
    //       segments[i].meta.SegmentLabel.includes("scribbles") &&
    //       segments[i].id == id
    //     ) {
    //       id = null;
    //       break;
    //     }
    //   }
    // },
    // async onSegmentation(
    //   file,
    //   labels,
    //   operation,
    //   slice,
    //   overlap,
    //   selectedIndex = 0
    // ) {
    //   // if (!selectedIndex) {
    //   //   selectedIndex = this.getSelectedActiveIndex();
    //   // }
    //   const { image } = SegmentationReader.parseNrrdData(file);

    //   if (labels) {
    //     let i = 0;
    //     for (var label in labels) {
    //       if (Array.isArray(labels)) {
    //         label = labels[label];
    //       }

    //       if (label === "background") {
    //         console.debug("Ignore Background...");
    //         continue;
    //       }
    //       const resp = createSegment(
    //         this.element,
    //         label,
    //         "",
    //         getLabelColor(label),
    //         // true
    //         i === 0 ? !overlap : false
    //       );
    //       if (i === 0) {
    //         selectedIndex = resp;
    //       }
    //       i++;
    //     }
    //   }

    //   if (!operation && overlap) {
    //     operation = "overlap";
    //   }

    //   updateSegment(
    //     this.element,
    //     selectedIndex.labelmapIndex,
    //     selectedIndex.segmentIndex,
    //     image,
    //     this.numberOfFrames,
    //     operation,
    //     slice
    //   );
    // },
    // async handleSegFileChange(e) {
    //   try {
    //     const file = e.target.files[0];
    //     const reader = new FileReader();
    //     reader.onloadend = async (event) => {
    //       await this.onSegmentation(event.target.result, this.labels);
    //     };
    //     reader.readAsArrayBuffer(file);
    //   } catch (err) {
    //     console.log(err);
    //   }
    // },
    handleFileChange(e) {
      // Create a ORTHOGRAPHIC viewport
      const viewportInput = {
        viewportId: this.viewportId,
        type: Enums.ViewportType.ORTHOGRAPHIC,
        element: this.element,
        defaultOptions: {
          background: [0.2, 0, 0.2],
        },
      };

      this.renderingEngine.enableElement(viewportInput);
      // Get the ORTHOGRAPHIC  viewport that was created
      this.viewport = this.renderingEngine.getViewport(this.viewportId);

      this.toolGroup.addViewport(this.viewportId, this.renderingEngineId);

      const imageIds = [];
      Array.from(e.target.files).forEach((file) => {
        const imageId =
          cornerstoneDICOMImageLoader.wadouri.fileManager.add(file);
        imageIds.push(imageId);
      });
      this.numberOfFrames = imageIds.length;
      this.loadAndViewImage(imageIds);
    },
    async loadAndViewImage(imageIds) {
      this.imageIds = imageIds;
      await prefetchMetadataInformation(imageIds);
      const volume = await volumeLoader.createAndCacheVolume(this.volumeId, {
        imageIds: this.imageIds,
      });
      volume.load();
      setVolumesForViewports(
        this.renderingEngine,
        [{ volumeId: this.volumeId }],
        [this.viewportId]
      );

      this.renderingEngine.renderViewport(this.imageIds);
    },
  },
};
</script>
