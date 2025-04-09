import { useState, useRef, useEffect } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Card, CardContent } from "@/components/ui/card";
import { FileEdit, UploadCloud, ArrowLeft, ArrowRight, Download } from "lucide-react";
import { motion } from "framer-motion";
import * as pdfjsLib from "pdfjs-dist/build/pdf";
import { PDFDocument, rgb, StandardFonts } from "pdf-lib";
import "pdfjs-dist/web/pdf_viewer.css";

pdfjsLib.GlobalWorkerOptions.workerSrc = `//cdnjs.cloudflare.com/ajax/libs/pdf.js/${pdfjsLib.version}/pdf.worker.min.js`;

export default function PDFEditor() {
  const [file, setFile] = useState(null);
  const [pdfDoc, setPdfDoc] = useState(null);
  const [pageNumber, setPageNumber] = useState(1);
  const [numPages, setNumPages] = useState(0);
  const [editedTexts, setEditedTexts] = useState(() => JSON.parse(localStorage.getItem("editedTexts") || "{}"));
  const [replacedImages, setReplacedImages] = useState(() => JSON.parse(localStorage.getItem("replacedImages") || "{}"));
  const canvasRef = useRef(null);
  const textLayerRef = useRef(null);

  const handleFileChange = (e) => {
    const uploadedFile = e.target.files[0];
    setFile(uploadedFile);
  };

  const handleUpload = async () => {
    if (!file) return;
    const reader = new FileReader();
    reader.onload = async () => {
      const typedArray = new Uint8Array(reader.result);
      const loadingTask = pdfjsLib.getDocument(typedArray);
      const loadedPdf = await loadingTask.promise;
      setPdfDoc(loadedPdf);
      setNumPages(loadedPdf.numPages);
      setPageNumber(1);
      renderPage(loadedPdf, 1);
    };
    reader.readAsArrayBuffer(file);
  };

  const handleImageReplacement = (overlay, key) => {
    const input = document.createElement("input");
    input.type = "file";
    input.accept = "image/*";
    input.onchange = () => {
      const file = input.files[0];
      const reader = new FileReader();
      reader.onload = () => {
        overlay.style.backgroundImage = `url(${reader.result})`;
        overlay.style.backgroundSize = "cover";
        overlay.style.border = "none";
        overlay.style.transform = overlay.style.transform || "scaleY(-1)";
        setReplacedImages((prev) => {
          const updated = { ...prev, [key]: reader.result };
          localStorage.setItem("replacedImages", JSON.stringify(updated));
          return updated;
        });
      };
      reader.readAsDataURL(file);
    };
    input.click();
  };

  const renderPage = async (pdf, num) => {
    const page = await pdf.getPage(num);
    const viewport = page.getViewport({ scale: 1.5 });

    const canvas = canvasRef.current;
    const context = canvas.getContext("2d");
    canvas.height = viewport.height;
    canvas.width = viewport.width;

    const renderContext = {
      canvasContext: context,
      viewport: viewport,
    };
    await page.render(renderContext).promise;

    const opList = await page.getOperatorList();
    const imageOps = [];
    for (let i = 0; i < opList.fnArray.length; i++) {
      if (opList.fnArray[i] === pdfjsLib.OPS.paintImageXObject || opList.fnArray[i] === pdfjsLib.OPS.paintJpegXObject) {
        imageOps.push({ fn: opList.fnArray[i], args: opList.argsArray[i] });
      }
    }

    const textContent = await page.getTextContent();
    const textLayer = textLayerRef.current;
    while (textLayer && textLayer.firstChild) textLayer.removeChild(textLayer.firstChild);

    textContent.items.forEach((item, index) => {
      const span = document.createElement("span");
      span.textContent = editedTexts[`${num}-${index}`] || item.str;
      Object.assign(span.style, {
        position: "absolute",
        left: `${item.transform[4]}px`,
        top: `${viewport.height - item.transform[5]}px`,
        fontSize: `${item.transform[0]}px`,
        transform: `scaleX(${item.transform[0] / 10})`,
        cursor: "text",
        whiteSpace: "pre",
        fontFamily: "Helvetica, Arial, sans-serif",
      });
      span.classList.add("highlight-hover");
      span.contentEditable = true;
      span.oninput = () => {
        setEditedTexts((prev) => {
          const updated = { ...prev, [`${num}-${index}`]: span.textContent };
          localStorage.setItem("editedTexts", JSON.stringify(updated));
          return updated;
        });
      };
      textLayer.appendChild(span);
    });

    for (let i = 0; i < imageOps.length; i++) {
      const imageOp = imageOps[i];
      const transform = imageOp.args[1];
      const width = transform[0];
      const height = transform[3];
      const x = transform[4];
      const y = transform[5];
      const key = `${num}-${i}`;

      const overlay = document.createElement("div");
      Object.assign(overlay.style, {
        position: "absolute",
        left: `${x}px`,
        top: `${viewport.height - y}px`,
        width: `${width}px`,
        height: `${Math.abs(height)}px`,
        border: "2px dashed red",
        backgroundColor: "rgba(255, 0, 0, 0.05)",
        cursor: "pointer",
        transform: `scaleY(-1)`,
        transformOrigin: "top",
      });

      if (replacedImages[key]) {
        overlay.style.backgroundImage = `url(${replacedImages[key]})`;
        overlay.style.backgroundSize = "cover";
        overlay.style.border = "none";
      }

      overlay.title = "Click to replace image";
      overlay.onclick = () => handleImageReplacement(overlay, key);
      textLayer.appendChild(overlay);
    }
  };

  const downloadEditedPDF = async () => {
    if (!file) return;
    const reader = new FileReader();
    reader.onload = async () => {
      const existingPdfBytes = new Uint8Array(reader.result);
      const pdfDoc = await PDFDocument.load(existingPdfBytes);
      const helveticaFont = await pdfDoc.embedFont(StandardFonts.Helvetica);

      for (const [key, text] of Object.entries(editedTexts)) {
        const [pageIndexStr, itemIndex] = key.split("-");
        const pageIndex = parseInt(pageIndexStr, 10) - 1;
        const page = pdfDoc.getPages()[pageIndex];
        page.drawText(text, {
          x: 50,
          y: 700 - parseInt(itemIndex, 10) * 15,
          size: 12,
          font: helveticaFont,
          color: rgb(0, 0, 0),
        });
      }

      for (const [key, dataUrl] of Object.entries(replacedImages)) {
        const [pageIndexStr] = key.split("-");
        const pageIndex = parseInt(pageIndexStr, 10) - 1;
        const page = pdfDoc.getPages()[pageIndex];
        const pngImageBytes = await fetch(dataUrl).then((res) => res.arrayBuffer());
        const pngImage = await pdfDoc.embedPng(pngImageBytes);
        const { width, height } = pngImage.scale(0.5);
        page.drawImage(pngImage, {
          x: 100,
          y: 500,
          width,
          height,
        });
      }

      const pdfBytes = await pdfDoc.save();
      const blob = new Blob([pdfBytes], { type: "application/pdf" });
      const link = document.createElement("a");
      link.href = URL.createObjectURL(blob);
      link.download = "edited.pdf";
      link.click();
    };
    reader.readAsArrayBuffer(file);
  };

  useEffect(() => {
    if (pdfDoc) renderPage(pdfDoc, pageNumber);
  }, [pdfDoc, pageNumber]);

  const goToPrevPage = () => {
    if (pageNumber > 1) setPageNumber(pageNumber - 1);
  };

  const goToNextPage = () => {
    if (pageNumber < numPages) setPageNumber(pageNumber + 1);
  };

  return (
    <div className="p-6 max-w-5xl mx-auto">
      <style>
        {`
          .highlight-hover:hover {
            background-color: rgba(255, 255, 0, 0.4);
          }
        `}
      </style>
      <motion.h1
        initial={{ opacity: 0, y: -20 }}
        animate={{ opacity: 1, y: 0 }}
        className="text-3xl font-bold mb-6 text-center"
      >
        PDF Text & Image Editor
      </motion.h1>

      <Card className="mb-6">
        <CardContent className="p-4 flex items-center gap-4">
          <Input type="file" accept="application/pdf" onChange={handleFileChange} />
          <Button onClick={handleUpload} disabled={!file}>
            <UploadCloud className="mr-2 h-4 w-4" /> Upload PDF
          </Button>
          <Button onClick={downloadEditedPDF} disabled={!file}>
            <Download className="mr-2 h-4 w-4" /> Download Edited PDF
          </Button>
        </CardContent>
      </Card>

      <Card className="mb-4">
        <CardContent className="flex justify-between items-center">
          <Button onClick={goToPrevPage} disabled={pageNumber <= 1}>
            <ArrowLeft className="mr-2 h-4 w-4" /> Previous
          </Button>
          <p className="text-sm text-gray-700">
            Page {pageNumber} of {numPages || "..."}
          </p>
          <Button onClick={goToNextPage} disabled={pageNumber >= numPages}>
            Next <ArrowRight className="ml-2 h-4 w-4" />
          </Button>
        </CardContent>
      </Card>

      <Card className="min-h-[400px] relative">
        <CardContent className="flex justify-center items-center">
          {file ? (
            <div className="relative">
              <canvas ref={canvasRef} className="border shadow rounded-lg" />
              <div
                ref={textLayerRef}
                className="absolute top-0 left-0 w-full h-full pointer-events-auto"
              ></div>
            </div>
          ) : (
            <p className="text-gray-500">Upload a PDF to begin editing text and images.</p>
          )}
        </CardContent>
      </Card>
    </div>
  );
}
